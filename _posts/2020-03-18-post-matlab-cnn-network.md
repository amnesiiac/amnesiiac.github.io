---
layout: post
title: "convolution neural network infer & bp (neural networks, matlab proto)"
author: "melon"
date: 2020-03-18 19:26
categories: "2020"
tags:
  - neural network
  - matlab
  - math
  - todo
---

this article mainly covers the convolutional neural network infer & back-propagation algorithm.

<hr>

### # project files
1 LeNet_5.m: main script for train & test process.
```text
% 117082910078-melon

% --- structure of LeNet_5 ---
% input layer -> 28*28
% convolution layer1 -> 24*24*20
% rectified function = tanh()
% pooling layer1 -> 12*12*20
% full-connected layer -> 10010
% softmax layer -> (softmax 10 for digital numbers)

% --- dataset info ---
% training dataset = 800x10 samples
% testing dateset = 200x10 samples

clear all;
clc;

disp('preprocessing data......')                                               % === data preprocess ===
total_data = load('/Users/mac/Desktop/x/LeNet-5/mnist_data/mnist.mat');
num_sample_to_get_train = 80;
num_sample_to_get_test = 20;
train_data_0 = double(total_data.train0);
train_data_1 = double(total_data.train1);
train_data_2 = double(total_data.train2);
train_data_3 = double(total_data.train3);
train_data_4 = double(total_data.train4);
train_data_5 = double(total_data.train5);
train_data_6 = double(total_data.train6);
train_data_7 = double(total_data.train7);
train_data_8 = double(total_data.train8);
train_data_9 = double(total_data.train9);
test_data_0 = double(total_data.test0);
test_data_1 = double(total_data.test1);
test_data_2 = double(total_data.test2);
test_data_3 = double(total_data.test3);
test_data_4 = double(total_data.test4);
test_data_5 = double(total_data.test5);
test_data_6 = double(total_data.test6);
test_data_7 = double(total_data.test7);
test_data_8 = double(total_data.test8);
test_data_9 = double(total_data.test9);
train_data_0 = train_data_0(1:num_sample_to_get_train,:);
train_data_1 = train_data_1(1:num_sample_to_get_train,:);
train_data_2 = train_data_2(1:num_sample_to_get_train,:);
train_data_3 = train_data_3(1:num_sample_to_get_train,:);
train_data_4 = train_data_4(1:num_sample_to_get_train,:);
train_data_5 = train_data_5(1:num_sample_to_get_train,:);
train_data_6 = train_data_6(1:num_sample_to_get_train,:);
train_data_7 = train_data_7(1:num_sample_to_get_train,:);
train_data_8 = train_data_8(1:num_sample_to_get_train,:);
train_data_9 = train_data_9(1:num_sample_to_get_train,:);
test_data_0 = test_data_0(1:num_sample_to_get_test,:);
test_data_1 = test_data_1(1:num_sample_to_get_test,:);
test_data_2 = test_data_2(1:num_sample_to_get_test,:);
test_data_3 = test_data_3(1:num_sample_to_get_test,:);
test_data_4 = test_data_4(1:num_sample_to_get_test,:);
test_data_5 = test_data_5(1:num_sample_to_get_test,:);
test_data_6 = test_data_6(1:num_sample_to_get_test,:);
test_data_7 = test_data_7(1:num_sample_to_get_test,:);
test_data_8 = test_data_8(1:num_sample_to_get_test,:);
test_data_9 = test_data_9(1:num_sample_to_get_test,:);
                                                                               % === concat the train & test matrix ===
preprocessed_train_data = cat(1,train_data_0,train_data_1,train_data_2,...
    train_data_3,train_data_4,train_data_5,train_data_6,train_data_7,...
    train_data_8,train_data_9);
preprocessed_test_data = cat(1,test_data_0,test_data_1,test_data_2,...
    test_data_3,test_data_4,test_data_5,test_data_6,test_data_7,...
    test_data_8,test_data_9);

layer_c1_num = 20;                                                             % init LeNet structure
layer_s1_num = 20;
layer_f1_num = 100;
layer_output_num = 10;
yita = 0.01;                                                                   % learning rate
bias_c1 = (2*rand(1,20)-ones(1,20))/sqrt(20);                                  % init biases
bias_f1 = (2*rand(1,100)-ones(1,100))/sqrt(20);
[kernel_c1,kernel_f1] = init_kernel(layer_c1_num,layer_f1_num);                % init conv kernels
pooling_a = ones(2,2)/4;                                                       % init kernel for pooling
weight_f1 = (2*rand(20,100)-ones(20,100))/sqrt(20);                            % init weights for fc-layers
weight_output = (2*rand(100,10)-ones(100,10))/sqrt(100);
disp('network successfully initialized......');

disp('training networks......');                                               % === train process ===
tic

for iter=1:20                                                                  % for each epoch
    for n=1:num_sample_to_get_train                                            % each num in [0,9] has 800 train sample
        for m=0:9
            train_data = reshape(preprocessed_train_data(m*num_sample_to_get_train+n,:),[28,28]);  % reshape data
            % data normalize
            % train_data = wipe_off_average(train_data);

            for k=1:layer_c1_num                                                          % === inference ===
                state_c1(:,:,k) = convolution(train_data,kernel_c1(:,:,k));
                state_c1(:,:,k) = tanh(state_c1(:,:,k)+bias_c1(1,k));                     % rectified layer
                state_s1(:,:,k) = pooling(state_c1(:,:,k),pooling_a);                     % pooling layer
            end

            [state_f1_pre,state_f1_temp] = convolution_f1(state_s1,kernel_f1,weight_f1);  % full-connected layer
            
            for nn=1:layer_f1_num                                                         % rectifier layer tanh()
                state_f1(1,nn) = tanh(state_f1_pre(:,:,nn)+bias_f1(1,nn));
            end

            for nn=1:layer_output_num                                                     % softmax layer
                output(1,nn) = exp(state_f1*weight_output(:,nn))/sum(exp(state_f1*weight_output));
            end

            Error_cost=-output(1,m+1);                                                    % compute train loss
            % if (Error_cost<-0.98)
                % break;
            % end
                                                                                          % === bp process ===
            [kernel_c1,kernel_f1,weight_f1,weight_output,bias_c1,bias_f1]=...
            CNN_upweight(yita,Error_cost,m,train_data,...
                         state_c1,state_s1,...
                         state_f1,state_f1_temp,...
                         output,...
                         kernel_c1,kernel_f1,weight_f1,...
                         weight_output,bias_c1,bias_f1);
        end
    end
end
toc
time_for_train=toc;

disp('train steps completed,starting test procedure......');                          % === test process ===
count=0;                                                                              % num of correct detected samples
tic
for n=1:num_sample_to_get_test
    for m=0:9
        test_data = reshape(preprocessed_test_data(m*num_sample_to_get_test+n,:),[28,28]);  % samples for test
        % train_data = get_average(train_data);
        for k=1:layer_c1_num                                                                % convolution layer-1
            state_c1(:,:,k) = convolution(test_data,kernel_c1(:,:,k));
            state_c1(:,:,k) = tanh(state_c1(:,:,k)+bias_c1(1,k));                           % reactifier layer tanh()
            state_s1(:,:,k) = pooling(state_c1(:,:,k),pooling_a);                           % pooling layer-1
        end
        
        [state_f1_pre,state_f1_temp] = convolution_f1(state_s1,kernel_f1,weight_f1);        % full-connected layer

        for nn=1:layer_f1_num                                                               % recitifier layer
            state_f1(1,nn) = tanh(state_f1_pre(:,:,nn)+bias_f1(1,nn));
        end

        for nn=1:layer_output_num                                                           % softmax layer
            output(1,nn) = exp(state_f1*weight_output(:,nn))/sum(exp(state_f1*weight_output));
        end

        [p,classify] = max(output);
        if (classify==m+1)
            count=count+1;
        end
        fprintf('real num: %d  model output: %d  confidence level: %d \n',m,classify-1,p);
    end
end

toc
time_for_test=toc;
fprintf('time for train: %d seconds time for test: %d seconds',time_for_train,time_for_test);
model_accuracy=count/num_sample_to_get_test;
fprintf('the total accuracy of this model on test data set is %d',model_accuracy);
```

2 convolutional fully network layer:
```text
% === convolution 2d ===
% mention <m> is from 1 to 28-5+1:
% this means that this conv is like:
%              x x x x x                o o o x x     
%              x x x x x                o o o x x      m x x 
% primary img: x x x x x  conv kernel:  o o o x x ->   x x x  ...
%              x x x x x                x x x x x      x x x 
%              x x x x x                x x x x x     
%
% but not like: then m from 1 to 28
%                                      o o o
%              x x x x x               o o o x x x    m x x x x 
%              x x x x x               o o o x x x    x x x x x
% primary img: x x x x x  conv kernel:   x x x x x -> x x x x x ...
%              x x x x x                 x x x x x    x x x x x
%              x x x x x                 x x x x x    x x x x x

function [state] = convolution(data, kernel)
    [data_row,data_col] = size(data);
    [kernel_row,kernel_col] = size(kernel);
    for m=1:data_col-kernel_col+1
        for n=1:data_row-kernel_row+1
            state(m,n) = sum(sum(data(m:m+kernel_row-1, n:n+kernel_col-1).*kernel));
        end
    end
end
```

3 fully-connected layer inference
```text
function [state_f1,state_f1_temp]=convolution_f1(state_s1,kernel_f1,weight_f1)
    layer_f1_num=size(weight_f1,2);
    layer_s1_num=size(weight_f1,1);
     
    for n=1:layer_f1_num
        count=0;
        for m=1:layer_s1_num
            temp=state_s1(:,:,m)*weight_f1(m,n);
            count=count+temp;
        end
        state_f1_temp(:,:,n)=count;
        state_f1(:,:,n)=convolution(state_f1_temp(:,:,n),kernel_f1(:,:,n));
    end
end
```

4 down-sampling pooling layer implementation:
```text
function state = pooling(data,pooling_a)
    [data_row,data_col] = size(data);
    [pooling_row,pooling_col] = size(pooling_a);
    for m=1:data_col/pooling_col
        for n=1:data_row/pooling_row
            state(m,n) = sum(sum(data(2*m-1:2*m, 2*n-1:2*n).*pooling_a));
        end
    end
end
```

5 kernel initialization using randint:
```text
function [kernel_c1,kernel_f1]=init_kernel(layer_c1_num,layer_f1_num)
    for n=1:layer_c1_num
        kernel_c1(:,:,n) = (2*rand(5,5)-ones(5,5))/12;
    end
    for n=1:layer_f1_num
        kernel_f1(:,:,n) = (2*rand(12,12)-ones(12,12));
    end
end
```

6 convoluation network layer back-propagation function:
```text
function [kernel_c1, kernel_f1, weight_f1, weight_output, bias_c1, bias_f1] = ...
          CNN_upweight(yita, Error_cost, classify, train_data,...
          state_c1, state_s1, state_f1, state_f1_temp,...
          output,kernel_c1,kernel_f1,weight_f1,weight_output,bias_c1,bias_f1)
    layer_c1_num = size(state_c1, 3);                                          % number of nodes
    layer_s1_num = size(state_s1, 3);
    layer_f1_num = size(state_f1, 2);
    layer_output_num = size(output, 2);

    [c1_row, c1_col, ~] = size(state_c1);
    [s1_row, s1_col, ~] = size(state_s1);

    [kernel_c1_row, kernel_c1_col] = size(kernel_c1(:,:,1));
    [kernel_f1_row, kernel_f1_col] = size(kernel_f1(:,:,1));

    kernel_c1_temp = kernel_c1;                                                % save the weights of network
    kernel_f1_temp = kernel_f1;
    weight_f1_temp = weight_f1;
    weight_output_temp = weight_output;

    label = zeros(1, layer_output_num);                                        % error computing
    label(1, classify+1) = 1;
    delta_layer_output = output-label;

    for n=1:layer_output_num                                                   % update weight_output
        delta_weight_output_temp(:,n)=delta_layer_output(1,n)*state_f1';
    end
    weight_output_temp = weight_output_temp-yita*delta_weight_output_temp;

    for n=1:layer_f1_num                                                       % update bias_f1 & kernel_f1
        count=0;
        for m=1:layer_output_num
            count=count+delta_layer_output(1,m)*weight_output(n,m);
        end
        delta_layer_f1(1,n)=count*(1-tanh(state_f1(1,n)).^2);                  % bias_f1
        delta_bias_f1(1,n)=delta_layer_f1(1,n);
        delta_kernel_f1_temp(:,:,n)=delta_layer_f1(1,n)*state_f1_temp(:,:,n);  % kernel_f1
    end
    bias_f1=bias_f1-yita*delta_bias_f1;
    kernel_f1_temp=kernel_f1_temp-yita*delta_kernel_f1_temp;

    for n=1:layer_f1_num                                                       % update weight_f1
        delta_layer_f1_temp(:,:,n)=delta_layer_f1(1,n)*kernel_f1(:,:,n);
    end
    for n=1:layer_s1_num
        for m=1:layer_f1_num
            delta_weight_f1_temp(n,m)=sum(sum(delta_layer_f1_temp(:,:,m).*state_s1(:,:,n)));
        end
    end
    weight_f1_temp=weight_f1_temp-yita*delta_weight_f1_temp;

    for n=1:layer_s1_num                                                       % update bias_c1
        count=0;
        for m=1:layer_f1_num
            count=count+delta_layer_f1_temp(:,:,m)*weight_f1(n,m);   
        end
        delta_layer_s1(:,:,n)=count;
        delta_layer_c1(:,:,n)=kron(delta_layer_s1(:,:,n),ones(2,2)/4).*(1-tanh(state_c1(:,:,n)).^2);
        delta_bias_c1(1,n)=sum(sum(delta_layer_c1(:,:,n)));
    end
    bias_c1=bias_c1-yita*delta_bias_c1;

    for n=1:layer_c1_num                                                       % update kernel_c1
        delta_kernel_c1_temp(:,:,n)=rot90(conv2(train_data,rot90(delta_layer_c1(:,:,n),2),'valid'),2);
    end
    kernel_c1_temp=kernel_c1_temp-yita*delta_kernel_c1_temp;

    kernel_c1=kernel_c1_temp;                                                  % update weights of network
    kernel_f1=kernel_f1_temp;
    weight_f1=weight_f1_temp;
    weight_output=weight_output_temp;
end
```

<hr>

### # reference
https://blog.csdn.net/u010540396/article/details/52895074
