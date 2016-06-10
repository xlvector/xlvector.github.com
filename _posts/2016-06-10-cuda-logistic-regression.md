---
layout: post
title: 用CUDA实现一个稀疏的Logistic Regression
categories: cuda
tags: cuda gpu lr
---

最近在研究Mxnet，准备从底层先尝试一遍GPU的编程，这样更容易熟悉mshadow的那部分逻辑。于是用CUDA尝试了一下写一个稀疏的Logistic Regression的程序。所谓稀疏的LR，指的是特征非常多，而每个样本的特征非常少。因为是学习CUDA，所以我构造了一个问题，假设特征的个数是100W，而每个样本的特征只有100个左右。如果样本的特征ID中偶数的多于奇数的，我们就认为是正样本，否则是负样本。如下的代码用来产生一个样本：

    void mock_sample(const int max_feature_id, vector< pair<int, float> > & out, int * label) {
      int count = rand() % 100 + 10;
      int ret = 0;
      for(int i = 0; i < count; i++) {
        int fid = rand() % max_feature_id;
        if(fid % 2 == 0) ret += 1;
        else ret -= 1;
        if(abs(ret) > 10) break;
        out.push_back(make_pair<int, float>(fid, 1.0));
      }
      *label = (ret > 0) ? 1 : 0;
    }

LR的解法有很多种，PRML那本书上就提到了不下5种LR的优化方法。因为只是为了学习CUDA，我选择了最简单的一种，也就是基于mini-batch的梯度下降法。LR的优化核心主要是去学习w矩阵。而所谓稀疏性来自于计算点积<w, x>，这里w是一个100W维的稠密向量，而x是一个100维左右的稀疏向量。因此<w,x>就是一个稠密向量和稀疏向量的点击。如果我们考虑mini-batch, 假设X是由n个样本组成的一个batch，那么X就是一个n*100W的稀疏矩阵。而我们需要计算Xw就是一个稀疏矩阵和稠密向量的乘法。

为了解决这个乘法，我们第一个想到的是用 (cusparse)[http://docs.nvidia.com/cuda/cusparse/]，这是由CUDA提供的一个稀疏矩阵的计算库，在level2的函数中有CSR格式的稀疏矩阵和稠密向量的乘法：

    cusparseStatus_t cusparseScsrmv(cusparseHandle_t handle, 
            cusparseOperation_t transA, int m, int n, int nnz, 
            const float *alpha, const cusparseMatDescr_t descrA, 
            const float *csrValA, const int *csrRowPtrA, 
            const int *csrColIndA, const float *x, 
            const float *beta, float *y)

我首先用cusparse实现了一个版本，具体代码见 (dot_cusparse.cu)[https://github.com/xlvector/learning-dl/blob/master/cuda/lr/dot_cusparse.cu]。不过很不幸的是实现出来的速度还不如CPU的版本。所以我也没继续做优化，就放弃了CUSPARSE，准备完全自己实现。（按照后来的经验，如果继续优化，是有可能优化的比较快的，所以读者可以试一试）。

后来我想，其实如果用COO的表示方法，其实可以很简单的直接实现Xw的乘法，代码如下：

    __global__ void dot(float * val, int *row_ind, int *col_ind, int nnz, float * ret, float * w) {
      const int tid = (blockIdx.x * blockDim.x) + threadIdx.x;
      if (tid < nnz) {
        int r = row_ind[tid];
        int c = col_ind[tid];
        float v = val[tid];
        atomicAdd(&ret[r], v * w[c]);
      }
    }

这里，val, row_ind, col_ind 是矩阵X的COO表示：

    X[row_ind[i]][col_ind[i]] = val[i]
    nnz = len(val) 是X中非0元素的个数

ret是dot的返回，也就是Xw的计算结果，是1个n维的向量，下面一步就是计算这个n维向量中每一个元素的sigmoid：

    __global__ void vec_sigmoid(float * d, int num) {
      const int tid = (blockIdx.x * blockDim.x) + threadIdx.x;
      if (tid < num) {
        if(d[tid] > 10.0) d[tid] = 1.0;
        else if(d[tid] < -10.0) d[tid] = 0.0;
        else d[tid] = 1.0 / (1.0 + exp(-1.0 * d[tid]));
      }
    }

最后一步就是更新w向量，这里我们不考虑正则化，函数如下：

    __global__ void grad(float * val, int * row_ind, int *col_ind, float * mat_err,
                 int nnz, float *act, float *label, 
                 float *w, float learning_rate) {
      const int tid = (blockIdx.x * blockDim.x) + threadIdx.x;
      if (tid < nnz) {
        int r = row_ind[tid];
        int c = col_ind[tid];
        float v = val[tid];
        //mat_err是为了后面打印训练集的误差而纪录的
        mat_err[tid] = abs(label[r] - act[r]);
        float err = v * (label[r] - act[r]);
        atomicAdd(&w[c], learning_rate * err);
      }
    }

整个LR的函数如下：

    void lr(const vector< vector< pair<int, float> > > & data, 
        const vector<float> & label,
        CooMatrixHost * coo_mat_host, 
        CooMatrix * coo_mat,
        float * w, int ncol, int batch) {
      vec2coo(data, coo_mat_host, coo_mat);
      CUDA_CALL(cudaMemcpyAsync(coo_mat->label, label.data(), sizeof(float) * label.size(), cudaMemcpyHostToDevice, stream));
      CUDA_CALL(cudaMemset(coo_mat->act, 0, sizeof(float) * data.size()));

      int shared_memory_usage = 1;
      int num_blocks = ((coo_mat->nnz + (NUM_THREADS - 1)) / NUM_THREADS);
      dot<<<num_blocks, NUM_THREADS, shared_memory_usage, stream>>>(coo_mat->val,
                                    coo_mat->row_ind,
                                    coo_mat->col_ind,
                                    coo_mat->nnz, 
                                    coo_mat->act, w);
      
      num_blocks = ((data.size() + (NUM_THREADS - 1)) / NUM_THREADS);
      vec_sigmoid<<<num_blocks, NUM_THREADS, shared_memory_usage, stream>>>(coo_mat->act, data.size());
      
      num_blocks = ((coo_mat->nnz + (NUM_THREADS - 1)) / NUM_THREADS);
      grad<<<num_blocks, NUM_THREADS, shared_memory_usage, stream>>>(coo_mat->val,
                                     coo_mat->row_ind,
                                     coo_mat->col_ind,
                                     coo_mat->err,
                                     coo_mat->nnz, 
                                     coo_mat->act,
                                     coo_mat->label, 
                                     w, 0.01);
      if (batch % 10000 == 0){
        float * err = (float*) malloc(sizeof(float) * coo_mat->nnz);
        CUDA_CALL(cudaMemcpyAsync(err, coo_mat->err, sizeof(float) * coo_mat->nnz, cudaMemcpyDeviceToHost, stream));
        float total = 0.;
        for(int i = 0; i < coo_mat->nnz; i++) total += err[i];
        cout << total / (float) coo_mat->nnz << endl;
      }
    }

整个程序并不复杂，所有代码见(这里)[https://github.com/xlvector/learning-dl/blob/master/cuda/lr/dot.cu]。这里也没有考虑多线程和多卡。其中优化的点有几个：

    1. 样本是在CPU内存中生成的，而计算是在GPU中完成的。所以就涉及到CPU向GPU内存拷贝的问题。我们是在vec2coo函数中完成这一步的。
    2. 因为不考虑多线程和多卡，因此CPU的内存可以预先分配好（zeroCooMatrixHost)。GPU的内存也可以事先分配好(zeroCooMatrix)。否则malloc和cudaMalloc将是最耗时的函数。
    3. 使用stream，cudaMemoryAsync。

这里还有一些没有考虑到的优化点，后面需要再试试。

    1. 多线程
    2. 多个stream
    3. 多个卡

因为刚开始写CUDA的程序，读者发现这个代码有任何问题请发issue告诉我。


