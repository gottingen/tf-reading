tf 1.x与tf 2.6变化
===

一般来说在tf 2.x中都可以使用tf.compat.v1.xxxxxx来替代原来tf 1.x中的tf.xxxxxx

* 
    -> 

* tf 1.x： -> tf 2.6：

| tf1.x | tf2.6| 备注 |
| :----: | :----: | :----: |
| tf.contrib.layers.xavier_initializer | tf.keras.initializers.glorot_normal | |
| tf.contrib.layers | tf.keras.layers |  |
| from tensorflow.contrib import tpu | from tensorflow import tpu |  |
| from tensorflow.contrib import data | from tensorflow import data |  |
| from tensorflow.contrib import metrics |from tensorflow import metrics | |
| from tensorflow.contrib import cluster_resolver |  from tensorflow.python.distribute import cluster_resolver | |
|tf.contrib.layers.layer_norm |  tf.keras.layers.LayerNormalization | 这两个函数实现不同，不能只换名字 |

| Keras 2.2.5 | Keras 2.6.0 | 备注 |
| :----: | :----: | :----: |
| from keras.optimizers import Adam | from keras.optimizer_v2.adam import Adam | |
| from keras.utils import to_categorical | from keras.utils.np_utils import to_categorical | |

