## 1. Training

| 함수명 | 기능 | 예제 |
| --- | --- | --- |
| SGDSolver\(\) | 학습 | `solver = caffe.SGDSolver(prototxt_solver)` |
|  |  |  |

일반적인 Train Loop코드 
```python
%%time
niter = 200
test_interval = 25
# losses will also be stored in the log
train_loss = zeros(niter)
test_acc = zeros(int(np.ceil(niter / test_interval)))
output = zeros((niter, 8, 10))

# the main solver loop
for it in range(niter):
    solver.step(1)  # SGD by Caffe
    
    # store the train loss
    train_loss[it] = solver.net.blobs['loss'].data
    
    # store the output on the first test batch
    # (start the forward pass at conv1 to avoid loading new data)
    solver.test_nets[0].forward(start='conv1')
    output[it] = solver.test_nets[0].blobs['score'].data[:8]
    
    # run a full test every so often
    # (Caffe can also do this for us and write to a log, but we show here
    #  how to do it directly in Python, where more complicated things are easier.)
    if it % test_interval == 0:
        print 'Iteration', it, 'testing...'
        correct = 0
        for test_it in range(100):
            solver.test_nets[0].forward()
            correct += sum(solver.test_nets[0].blobs['score'].data.argmax(1)
                           == solver.test_nets[0].blobs['label'].data)
        test_acc[it // test_interval] = correct / 1e4
```




## 2. Testing

| 함수명 | 기능 | 예제 |
| --- | --- | --- |
| SGDSolver\(\) | 학습 | `solver = caffe.SGDSolver(prototxt_solver)` |
|  |  |  |

### 2.1 사전 요구 파일 

* a caffemodel created during training needs
* a matching deploy .prototxt definition 

Both prerequisites are fulfilled when writing regular snapshots during training and using `tools.prototxt.train2deploy` on the generated `.prototxt` network definitions

### 2.2 Network 초기화 

```python
net = caffe.Net(deploy_prototxt_path, caffemodel_path, caffe.TEST)
```
- `caffe.TEST` : use test mode (e.g., don't perform dropout)


### 2.3 입력 파일 설정

 The input data can then be set by reshaping the data blob:

```python
image = cv2.imread(image_path)
net.blobs['data'].reshape(1, image.shape[2], image.shape[0], image.shape[1])
```

* `caffe.Net` is the central interface for loading, configuring, and running models. -

* `caffe.Classifier` and `caffe.Detector` provide convenience interfaces for common tasks.

* `caffe.SGDSolver` exposes the solving interface.

* `caffe.io` handles input / output with preprocessing and protocol buffers.

* `caffe.draw` visualizes network architectures.

* Caffe blobs are exposed as numpy ndarrays for ease-of-use and efficiency.


## 3. Fine tuning 

> [fine-tuning.ipynb](http://nbviewer.jupyter.org/github/BVLC/caffe/blob/tutorial/examples/03-fine-tuning.ipynb)





## 4. 이미지 전처리 
Before checking the net let's define an input pre-processor, transformer, to help feed an image into the net.

```python


# configure input pre-processing
mu = np.load(caffe_root + 'python/caffe/imagenet/ilsvrc_2012_mean.npy')
mu = mu.mean(1).mean(1)  # average over pixels to obtain the mean (BGR) pixel values
transformer = caffe.io.Transformer({'data': net.blobs['data'].data.shape})
transformer.set_transpose('data', (2,0,1))  # move image channels to outermost dimension
transformer.set_mean('data', mu)            # subtract the dataset-mean value in each channel
transformer.set_raw_scale('data', 255)      # rescale from [0, 1] to [0, 255]
transformer.set_channel_swap('data', (2,1,0))  # swap channels from RGB to BGR
print 'Configured input.'

```
이후 작업 코드 

```python 
# set the size of the input (we can skip this if we're happy
net.blobs['data'].reshape(50,        # batch size
                          3,         # 3-channel (BGR) images
                          227, 227)  # image size is 227x227

# download an image
image = caffe.io.load_image(caffe_root + 'examples/images/coffee.jpg')
transformed_image = transformer.preprocess('data', image)
plt.imshow(image)
```

###### [참고] OpenCV 3.3.0버젼부터 DNN지원 
```python 
from cv2 import dnn
blob = dnn.blobFromImage(cv2.imread('space_shuttle.jpg'), 1, (224, 224), (104, 117, 123))
```


---

## pycaffe interface

```python
caffe.set_device(0)

caffe.set_mode_gpu()

solver = caffe.SGDSolver('PATH/TO/THE/SOLVER.PROTOTXT')

solver.net.blobs.items()
solver.net.params.items() 

solver.test_nets[0].forward() # test net (there can be more than one)

# forward one time of minibatch of SGD
solver.step(1)

# useful for visualizing input data
imshow(solver.net.blobs['data'].data[:8, 0].transpose(1,0,2).reshape(28, 8*28), cmap='gray'); axis('off')

# useful for visualizing filter ( display 5x5 filter 4x5 tiles)
imshow(solver.net.params['conv1'].diff[:,0].reshape(4,5,5,5).transpose(0,2,1,3).reshape(4*5, 5*5), cmap='gray'; axis('off')


train_loss[it] = solver.net.blobs['loss'].data


# (start the forward pass at conv1 to avoid loading new data)
    solver.test_nets[0].forward(start='conv1')

solver.test_nets[0].blobs['score'].data.argmax(1)
              == solver.test_nets[0].blobs['label'].data

# when using pre-trained model
imagenet_net = caffe.Net(imagenet_net_filename, weights, caffe.TEST)

or
solver.net.copy_from(pretrained_model)


imagenet_net.forward()


# if you want to check current number of iteration, 
solver.iter

# save net

net = self.solver.net
net.save(str(SAVE_PATH))
```



