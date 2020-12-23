---
title: "Digit Recognition with Convolutional Neural Networks"
date: 2020-12-11T11:45:50Z
draft: true

description: "bla"
summary: "bla bla"

tags: ["Convolutional Neural Networks", 'Deep Learning', 'MNIST', 'Digit Recognition']
categories: ["Deep Learning"]

featuredImage: ""

lightgallery: false
---

# Introduction

In this blog note, I give a quick introduction to convolutional neural networks (CNN), by building the most classical beginner-friendly project: Digit recognition.

## The problem : Digit recognition

The problem is simple. We want to build a model which has an image of a digit for input and outputs a 'guess' of what the digit is.

## A solution : Convolutional neural networks

## The tool : Keras  




```python
import tensorflow as tf
from tensorflow import keras
```

# The MNIST Database

## Import


```python
from tensorflow.keras import datasets
```


```python
(X_train, y_train), (X_test, y_test) = tf.keras.datasets.mnist.load_data()
```

## Visualise


```python
print('X_train shape = {}'.format(X_train.shape))
print('y_train shape = {}'.format(y_train.shape))
print('X_test shape = {}'.format(X_test.shape))
print('y_test shape = {}'.format(y_test.shape))
```
```
X_train shape = (60000, 28, 28)
y_train shape = (60000,)
X_test shape = (10000, 28, 28)
y_test shape = (10000,)
```


```python
import matplotlib.pyplot as plt
for i in range(1,10):
    plt.subplot(330 + i)
    plt.imshow(X_train[i-1], cmap=plt.get_cmap('gray_r'))
plt.show()
```


{{<style "text-align: center">}}
  
{{<figure src="output_7_0.png">}}

{{</style>}}


## Preparing the Data


```python
input_shape = (28,28,1)
num_classes = 10
```


```python
X_train.reshape(X_train.shape[0],28,28,1)
X_test.reshape(X_test.shape[0],28,28,1)

X_train = X_train/255
X_test = X_test/255

# Reserve 10,000 samples for validation
X_val = X_train[-10000:]
y_val = y_train[-10000:]
X_train = X_train[:-10000]
y_train = y_train[:-10000]
```

# The Model

## Building


```python
from tensorflow.keras import layers
```


```python
inputs = keras.Input(shape=(28,28,1), name="digits")
x = layers.Conv2D(32, kernel_size = (3,3), activation='relu')(inputs)
x = layers.MaxPool2D(pool_size = (2,2))(x)
x = layers.Conv2D(64, kernel_size = (3,3), activation='relu')(x)
x = layers.MaxPool2D(pool_size = (2,2))(x)
x = layers.Flatten()(x)
x = layers.Dropout(0.5)(x)
outputs = layers.Dense(10, activation="softmax", name="predictions")(x)

model = keras.Model(inputs=inputs, outputs=outputs)
```

## Training


```python
model.compile(
    optimizer=keras.optimizers.Adam(),
    loss=keras.losses.SparseCategoricalCrossentropy(),
    metrics=[keras.metrics.SparseCategoricalAccuracy()],
)
```


```python
history = model.fit(
    X_train,
    y_train,
    batch_size=128,
    epochs=15,
    validation_data=(X_val, y_val),
)
```
```
Epoch 1/15
391/391 [==============================] - 79s 202ms/step
- loss: 0.4529 
- sparse_categorical_accuracy: 0.8584 
- val_loss: 0.1151
- val_sparse_categorical_accuracy: 0.9682
Epoch 2/15
391/391 [==============================] - 83s 212ms/step
- loss: 0.1478
- sparse_categorical_accuracy: 0.9542
- val_loss: 0.0846
- val_sparse_categorical_accuracy: 0.9765
...
Epoch 15/15
391/391 [==============================] - 102s 261ms/step
- loss: 0.0495
- sparse_categorical_accuracy: 0.9840
- val_loss: 0.0374
- val_sparse_categorical_accuracy: 0.9890
```

## Evaluation


```python
plt.plot(list(range(1,16)), history.history['loss'])
plt.plot(list(range(1,16)), history.history['val_loss'])
plt.show()
```


{{<style "text-align: center">}}
{{<figure src="output_18_0.png">}}
{{</style>}}  



```python
plt.plot(list(range(1,16)), history.history['sparse_categorical_accuracy'])
plt.plot(list(range(1,16)), history.history['val_sparse_categorical_accuracy'])
plt.show()
```

{{<style "text-align: center">}}
{{<figure src="output_19_0.png">}}
{{</style>}}



```python
print("Evaluate on test data")
results = model.evaluate(X_test, y_test, batch_size=128)
print("test loss, test acc:", results)

print("Generate predictions")
predictions = model.predict(X_test)
print("predictions shape:", predictions.shape)
```
```
Evaluate on test data
79/79 [==============================] - 4s 55ms/step
- loss: 0.0321
- sparse_categorical_accuracy: 0.9895
test loss, test acc: [0.03210258483886719, 0.9894999861717224]
Generate predictions
predictions shape: (10000, 10)
```

