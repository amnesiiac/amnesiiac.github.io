---
layout: post
title: "fully connected neural network infer & bp (neural networks, matlab proto)"
author: "melon"
date: 2020-03-17 20:26
categories: "2020"
tags:
  - neural network
  - matlab
  - math
---

this article mainly covers the fully-connected neural network infer & back-propagation algorithm.

<hr>

### # preparations
the activation function for fc network model in this article are logistic function:

$$
f(x) = \frac{1}{1+e^{-x}}
$$

and the derivative format of the activation function is as:

$$
f'(x) = \frac{1}{1+e^{-x}} \cdot \frac{e^{-x}}{1+e^{-x}}
= \frac{1}{1+e^{-x}} \cdot (1-\frac{1}{1+e^{-x}})
= f(x) \cdot (1-f(x))
$$

in following part, this equation will be used by each bp partial derivative computation.

<hr>

### # case 1: a simple 1-8-1 fc model y=f(v.f(w.x)) inference & bp process to fit 1+x+x^2
1 fc network structure overview, with inference & back-propagation computation marked:
<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/fc1.pdf" width="650"/>

2 matlab implementation in details:
```text
% 117082910078-melon 2018-6-4
clc;
clear;
                                                        % === init basic parameters ===
TrainCount = 5000;                                      % train 10000 epochs
DataNum = 1001;                                         % data set 1001
Hide_Out = zeros(1,8);                                  % init hide layer out (train) = 1 x 8 matrix
Hide_OutTest = zeros(1,8);                              % init hide layer out (test) = 1 x 8 matrix
Rate_H = 0.02;                                          % hidden layer learning rate
Rate_O = 0.01;                                          % output layer learning rate

                                                        % === setup inputs x & label y ===
x = -5:0.01:6;                                          % x range with step: 1101
y = 1 + x + x.*x;                                       % the curve function to fit

                                                        % === init parameters & normalize data ===
x_Nor = (x - (-5) + 1)/(5 - (-5) + 1);                  % norm x as data: (x-xmin) / (xmax-xmin)
y_Nor = (y - 0.75 + 1)/(31 - 0.75 + 1);                 % norm y as label: (y-ymin) / (ymax-ymin)

w = 2*(rand(1,8)-0.5);                                  % init hiden layer weights with rand number in [-1,1]
v = 2*(rand(1,8)-0.5);                                  % init output layer weights with rand number in [-1,1]
dw = zeros(1,8);                                        % init hiden layer biases as 0
dv = zeros(1,8);                                        % init output layer biases as 0

                                                        % === seperate train & test set ===
xNum = length(x);                                       % total 1101 data
Itest = 1:11:xNum;                                      % itest=1,12,23...
Xtest_Nor = x_Nor(Itest);                               % x for test
Ytest_Nor = y_Nor(Itest);                               % y for test
Xtrain_Nor = x_Nor;                                     % x for train 
Ytrain_Nor = y_Nor;                                     % y for train
Xtrain_Nor(Itest) = [];                                 % del test data from whole dataset
Ytrain_Nor(Itest) = [];

Count = 0;
e2 = 0;
while(1)
    e1 = 0;                                                                         % === training part === %
    for i = 1:1000
        for j = 1:8                                                                 % fc network inference part 1
            sum1 = w(j)*Xtrain_Nor(i);                                              % hiden_layer=weight1*train_data
            Hide_Out(j) = 1/(1 + exp(-1*sum1));                                     % recitfy func: logistic
        end
        sum2 = 0;
        for k = 1:8                                                                 % fc network inference part 2
            sum2 = sum2 + v(k)*Hide_Out(k);                                         % sum2 = v1*out1+v2*out2+...v8*out8
        end

        outputdata = sum2;                                                          % output of fc network
        e1 = e1 + abs(outputdata - Ytrain_Nor(i));                                  % infer err: |nn_output-label|

        for s = 1:8                                                                 % back-propagation
            dv(s) = Rate_O * (Ytrain_Nor(i)-outputdata) * Hide_Out(s);              % update v, ref: bp blog (eq15)
            v(s) = v(s) + dv(s);
            alfa = 0;                                                               % update hiden layer weights
            alfa = alfa + (Ytrain_Nor(i)-outputdata) * v(s);                        % ref: bp blog (eq14)
            dw(s) = Rate_H * alfa * (Hide_Out(s)*(1-Hide_Out(s))) * Xtrain_Nor(i);  % ref: bp blog (eq14)
            w(s) = w(s) + dw(s);
        end
    end

    e11 = e1 / 1000;                                                % compute the amortized err of each train sample
    myErr1(Count+1) = e11; 

    e2 = 0;                                                         % === testing part === %
    trained_model_output_container = zeros(1,101);
    for p = 1:100                                                   % model inference on test data set
        for l = 1:8                                                 % first half
            sum3 = w(l)* Xtest_Nor(p);
            Hide_OutTest(l) =  1/(1 + exp(-1*sum3));                
        end
        sum4 = 0;
        for m = 1:8                                                 % second half
            sum4 = sum4 + v(m)*Hide_OutTest(m);
        end 
        OutputData_Test = sum4;                                     % output of whole test set
        e2 = e2 + abs(OutputData_Test - Ytest_Nor(p));              % total error of whole test set
        trained_model_output_container(p) = OutputData_Test;
    end

    e12 = e2 / 100;                                                 % e12: error on each sample
    myErr2(Count+1) = e12;                                          % record err on each epoch
    Count = Count + 1;                                              % epochs+1
    
    if mod(Count,10000)==0 || Count==TrainCount                     % === plotting part === %
        figure(1);
        hold on
        plot(1:Count,myErr1,'r',1:Count,myErr2,'g')
        title('train & test error on data set is')
        hold off
    end

    if(Count > TrainCount || e12 < 0.0001)                          % === iteration terminte condition === %
        break; 
    end
end

figure(2);                                                          % === plot fitting curve === %
hold on
x_new = Xtest_Nor(1:100);                                           % x
y_new = trained_model_output_container(1:100);                      % y: output by model inference
plot(x_Nor,y_Nor,'r');
plot(x_new,y_new,'b');
title('fc-net(1-n-1) to fit the curve')
xlabel('x');
ylabel('y');
hold off
```

3 program output graph for train/test loss curve, with test set fitting comparsion:
<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/fc4.pdf" width="650"/>

<hr>

### # case 2: a simple 1-8-8-1 fc model y=f(v.f(w.x)) inference & bp process to fit sin(x)
1 fc network structure overview, with inference & back-propagation computation marked:
<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/fc2.pdf" width="650"/>

2 matlab implementation in details:
```text
% 117082910078-melon 2018-6-26
clc;
clear all;

traincount = 20000;               % train 10000 epochs
batch_size = 20;                  % set batch size for weights refreshing
datanum = 1001;                   % data set 1001

rate_hiden_layer = 0.1;           % hiden layer learning rate
rate_output_layer = 0.05;         % output layer learning rate

x = -5:0.01:6;                    % derive sample data set is 1101 double numbers
y = sin(x);
y_handle_1 = @(x) sin(x);
y_handle_2 = @(x) -sin(x);


min = fminbnd(y_handle_1,-5,6);   % === init parameters & normalize data ===
minvalue = sin(min);
max = fminbnd(y_handle_2,-5,6);
maxvalue = sin(max);

normalized_x = (x-(-5)) / (6-(-5));
normalized_y = (y-minvalue) / (maxvalue - minvalue);

w = rand(8,1);                                              % init 8 hiden layer weights as rand number in [-1,1]
v = rand(8,8);                                              % init 64 output layer weights as rand number in [-1,1]
d_w = zeros(8,1);                                           % init hiden layer biases as 0 
d_v = zeros(8,8);                                           % init output layer biases as 0

                                                            % === seperate the train & test set ===
xNum = length(x);                                           % total 1101 data
index_test=1:11:xNum;                                       % itest=1,12,23...
normalized_test_x = normalized_x(index_test);               % derive corresponding data by the indexes
normalized_test_y = normalized_y(index_test);               % derive corresponding label by the indexes

normalized_train_x = normalized_x;                          % normalized_x_for_train
normalized_train_y = normalized_y;                          % normalized_y_for_train
normalized_train_x(index_test) = [];                        % cut down the test data from train data
normalized_train_y(index_test) = [];

train_capcibility = length(normalized_train_x);
test_capcibility = length(normalized_test_x);

count = 0;
e2 = 0;
error2 = 0;

while(1)
    e1 = 0;                                                 % === training process ===
    for i = 1:train_capcibility
        z2=w*normalized_train_x(i);                         % z2:(8,1)
        a2=1./(1 + exp(-z2));                               % rectifier: logistic; a2:(8,1)
        
        z3 = v*a2;                                          % z3=(8,8)*(8,1)=(8,1)
        a3 = 1./(1 + exp(-z3));                             % rectifier: logistic; a3:(8,1)

        logit = sum(a3);                                    % output = sum(a3) = (1)
        e1 = e1 + abs(logit - normalized_train_y(i));
        error2 = error2 + (logit - normalized_train_y(i));

                                                            % === back-propagation process ===
        k=ones(1,8);                                        % update w & b of output_layer & hidden_layer
        delta_4 = -(normalized_train_y(i) - logit)*1;       % delta_4=1x1  ref: bp blog (eq5)
        delta_3 = ((k*delta_4)').*(a3.*(1-a3));             % delta_3=8x1  ref: bp blog (eq14)
        delta_2 = ((v')*delta_3).*(a2.*(1-a2));             % delta_2=8x1  ref: bp blog (eq14)
        d_w = normalized_train_x(i)*delta_2;                % d_w = 8x1    ref: bp blog (eq15)
        d_v = a2*(delta_3');                                % d_v = 8x8    ref: bp blog (eq15)
        d_v = d_v';                                         % transpose d_v  we get the final right d_v
        v = v - (rate_hiden_layer * d_v);                   % new_weight = old_weight - alpha * d_weights
        w = w - (rate_output_layer * d_w);                  % new_weight = old_weight - alpha * d_weights
    end
                                                            % === error compute & curve plot ===
    e11 = e1 / train_capcibility;                           % average error on single data
    myerror1(count+1) = e11;
    count = count + 1;
    if count==traincount
        figure(1);
        hold on
        % plot(Count, e11, 'r', Count, e12, 'b');
        plot(1:count, myerror1, 'r')
    end

    if(count > traincount || e11 < 0.0001)                  % === training terminte condition ===
        break;
    end
end

e2 = 0;                                                     % === testing process ===
trained_model_output_container=zeros(1,100);
for p = 1:test_capcibility                                  % test data set capcibility = 100
    z2_test=w*normalized_test_x(p);                         % model infer on test set
    a2_test=1./(1 + exp(-z2_test));                         % rectifier: logistic; a1=f(z1) a1:(8,1)
        
    z3_test = v*a2_test;                                    % z3_test = (8,8)*(8,1) = (8,1)
    a3_test = 1./(1 + exp(-z3_test));                       % activated z3_test
    logit_test = sum(a3_test);                              % neural network model output
        
    e2 = e2 + abs(logit_test - normalized_test_y(p));       % test data set total error
    trained_model_output_container(p) = logit_test;
end
                                                            % === err computation & fit curve plot ===
e12 = e2 / test_capcibility;                                % avg error on single sample curve
plot(e12,'o')
title(['final average error on test set is ',num2str(e12)])
hold off

figure(2);                                                  % plot target & test fitting curve
hold on
x_new = normalized_test_x(1:test_capcibility);
y_new = trained_model_output_container(1:test_capcibility);
plot(normalized_test_x,normalized_test_y,'r');
% plot(x_new,y_new,'b');
size = 10;
scatter(x_new,y_new,size);
title('fc-net(1x8x8x1) to fit the curve')
xlabel('x');
ylabel('y');
hold off
```

3 program output graph for train/test loss curve, with test set fitting comparsion:
<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/fc5.pdf" width="650"/>

<hr>

### # case 3: a simple 1-8-8-1 fc model y=f(v.f(w.x+b1)+b2) inference & bp process to fit sin(x)
1 fc network structure overview, with inference & back-propagation computation marked:
<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/fc3.pdf" width="650"/>

2 matlab implementation in details:
```text
% 117082910078-melon 2018-6-26
clc;
clear all;

traincount = 10000;                                % train 10000 epochs
batch_size = 20;                                   % set batch size for weights refreshing
hiden_layer_units = 8;                             % n = hiden_layer_units = the number of hiden neurons

rate_hiden_layer = 0.1;                            % hiden layer learning rate
rate_input_layer = 0.05;                           % output layer learning rate

x = -5:0.01:6;                                     % derive sample data set is 1101 double numbers
y = sin(x);
y_handle_1 = @(x) sin(x);
y_handle_2 = @(x) -sin(x);

min = fminbnd(y_handle_1,-5,6);                    % step2: initialize parameters & normalize data
minvalue = y_handle_1(min);                        % y_handle_1(min) = sin(min)

max = fminbnd(y_handle_2,-5,6);
maxvalue = y_handle_1(max);                        % y_handle_1(max) = sin(max)

normalized_x = (x-(-5))/(6-(-5));
normalized_y = (y-minvalue)/(maxvalue - minvalue);   

w = rand(hiden_layer_units,1);                     % init 8 hiden layer weights as rand number in [-1,1]
b1 = rand(hiden_layer_units,1);                    % init biases

v = (rand(hiden_layer_units,hiden_layer_units));   % init 64 output layer weights as rand number in [-1,1]
b2 = rand(hiden_layer_units,1);                    % init biases
d_w = zeros(hiden_layer_units,1);                  % init hidden layer biases as 0
d_v = zeros(hiden_layer_units,hiden_layer_units);  % init output layer biases as 0

                                                   % === seperate the train & test set ===
xNum = length(x);                                  % total 1101 data
index_test = 1:11:xNum;                            % itest=1,12,23...
normalized_test_x = normalized_x(index_test);      % derive corresponding data by the indexes
normalized_test_y = normalized_y(index_test);      % derive corresponding label by the indexes

normalized_train_x = normalized_x;                 % norm x for train
normalized_train_y = normalized_y;                 % norm y for train
normalized_train_x(index_test) = [];               % cut down the test data from train data
normalized_train_y(index_test) = [];

train_capcibility = length(normalized_train_x);
test_capcibility = length(normalized_test_x);

count = 0;
e2 = 0;
error2 = 0;
while(1)
    e1 = 0;                                                   % === train process ===
    for i = 1:train_capcibility
        z2 = w*normalized_train_x(i)+b1;                      % z1=w*x: (8x1) normalized_train_x(i):(1,1) z2:(8,1)
        a2 = 1./(1 + exp(-z2));                               % rectifier: logistic; a2=f(z2) a2: (8x1)
        z3 = v*a2+b2;                                         % z3:(8,1); v[j][i]: weights from i to j
        a3 = 1./(1 + exp(-z3));                               % a3:(8,1)
        logit = sum(a3);
        e1 = e1 + abs(logit - normalized_train_y(i));         % get the error=|nn_output-label|+error
        % error2 = error2 + (logit - normalized_train_y(i));
                                                              % === back-propagation on batch size = 1 ===
        k = ones(1,hiden_layer_units);                        % output_layer & hidden_layer weights refreshing
        delta_4 = -(normalized_train_y(i) - logit)*1;         % delta_4=1x1  b-p matlab function(7)
        delta_3 = ((k*delta_4)').*(a3.*(1-a3));               % delta_3=nx1  b-p matlab function(9)
        delta_2 = ((v')*delta_3).*(a2.*(1-a2));               % delta_2: (nx1)
        d_w = normalized_train_x(i)*delta_2;                  % d_w = nx1 b-p matlab function(12)
        d_b1 = delta_2;                                       % d_b1: (nx1)
        d_v = a2*(delta_3');                                  % d_v: (nxn)
        d_v = d_v';                                           % transpose for b-p matlab function (3) (12)
        d_b2 = delta_3;                                       % d_b2: (8,1)
        v = v - (rate_hiden_layer * d_v);                     % new_v = old_v - alpha_v * d_v
        w = w - (rate_input_layer * d_w);                     % new_w = old_w - alpha_w * d_w
        b1 = b1 - (rate_input_layer) * d_b1;                  % new_biases = old_biases - alpha * d_b
        b2 = b2 - (rate_hiden_layer) * d_b2;                  % new_biases = old_biases - alpha * d_b
    end

    e11 = e1 / train_capcibility;                             % === compute the error & plot err curve ===
    myerror1(count+1) = e11; 
    count = count + 1;                                        % epoch++
    if count==traincount
        figure(1);
        hold on
        % plot(Count,e11,'r',Count,e12,'b');
        plot(1:count,myerror1,'r')
    end

    if(count > traincount || e11 < 0.0001)                    % === training terminte condition ===
        break; 
    end
end

e2=0;                                                         % === test process ===
trained_model_output_container=zeros(1,100);
for p = 1:test_capcibility % test data set capcibility = 100
    z2_test = w*normalized_test_x(p)+b1;                      % network model infer on test set
    a2_test = 1./(1 + exp(-z2_test));                         % rectifier: logistic; a1=f(z1) a1(nx1)
        
    % v_ji weights from i to j v=nxn a2=nx1 z3=nx1
    % v_ji row:j  column:i
    z3_test = v*a2_test+b2;
    a3_test = 1./(1 + exp(-z3_test));
    logit_test = sum(a3_test);
        
    e2 = e2 + abs(logit_test - normalized_test_y(p));         % test data set total error
    trained_model_output_container(p) = logit_test;
end

e12 = e2 / test_capcibility;                                  % plot average single sample err curve
plot(e12,'o')
title(['final average error on test set is ',num2str(e12)])
hold off

figure(2);                                                    % plot target & test fitting curve
hold on
x_new = normalized_test_x(1:test_capcibility);
y_new = trained_model_output_container(1:test_capcibility);
plot(normalized_test_x,normalized_test_y,'r');
% plot(x_new,y_new,'b');
size = 20;
scatter(x_new,y_new,size);
title('fc-net(1-n-n-1) to fit the curve')
xlabel('x');
ylabel('y');
hold off
```

3 program output graph for train/test loss curve, with test set fitting comparsion:
<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/fc6.pdf" width="650"/>
