# CoAudioText
A short tutorial for the co-utilization of audio and text data (multi-modal analysis)

## Contents
[0. Problem definition & loading dataset](https://github.com/warnikchow/coaudiotext/blob/master/README.md#0-problem-definition--loading-dataset)</br>
[1. Extracting acoustic features](https://github.com/warnikchow/coaudiotext/blob/master/README.md#1-extracting-acoustic-features)</br>
[2. Speech-only analysis with Librosa and Keras](https://github.com/warnikchow/coaudiotext/blob/master/README.md#2-speech-only-analysis-with-librosa-and-keras)</br>
[3. Self-attentive BiLSTM](https://github.com/warnikchow/coaudiotext/blob/master/README.md#3-self-attentive-bilstm)</br>
[4. Parallel utilization of audio and text data](https://github.com/warnikchow/coaudiotext/blob/master/README.md#4-parallel-utilization-of-audio-and-text-data)</br>
[5. Multi-hop attention](https://github.com/warnikchow/coaudiotext/blob/master/README.md#5-multi-hop-attention)</br>
[6. Cross-attention](https://github.com/warnikchow/coaudiotext/blob/master/README.md#6-cross-attention)

## 0. Problem definition & loading dataset

Understanding the intention of an utterance is challenging for some prosody-sensitive cases, especially when it is in the written form as in a text chatting or speech recognition output. **The main concern is detecting the directivity or rhetoricalness of an utterance and distinguishing the type of question.** Since it is inevitable to face both the issues regarding prosody and semantics, the identification is expected be benefited from the observations on human language processing mechanism. 

Challenging issues for spoken language understanding (SLU) modules include inferring the intention of syntactically ambiguous utterances. If an utterance has an underspecified sentence ender whose role is decided only upon prosody, the inference requires whole the acoustic and textual data of the speech for SLUs (and even human) to correctly infer the intention, since the pitch sequence, the duration between the words, and the overall tone decides the intention of the utterance. For example, in Seoul Korean which is *wh-in-situ*, many sentences incorporate various ways of interpretation that depend on the intonation, as shown in our [ICPhS paper](http://www.assta.org/proceedings/ICPhS2019/papers/ICPhS_3951.pdf).

Here, we attack the issue above utilizing the speech corpus that is distributed along with the paper. The scripts and speech files are available in [this github repository](https://github.com/warnikchow/prosem). As you download the folder from [the dropbox](https://www.dropbox.com/s/3tm6ylu21jpmnj8/ProSem_KOR_speech.zip?dl=0), unzip the folder in YOUR DIRECTORY so that you have *ProSem_KOR_speech* folder there. In it, there are the folders named *FEMALE* and *MALE* each containing 3,551 Korean speech utterances. Also, place the folder *text* in YOUR DIRECTORY, which contains the scripts of the speech files.

## 1. Extracting acoustic features

First, since we only utilize the utterances that can be disambiguated with speech, here we extract the acoustic features from the files. There are many ways to abstractize the physical components, but here we utilize [mel spectrogram](https://towardsdatascience.com/getting-to-know-the-mel-spectrogram-31bca3e2d9d0) due to its rich acoustic information and intuitive concept. The process is done with [Librosa](https://librosa.github.io/librosa/), a python-based audio signal processing library.

```python
def make_data(fname,fnum,shuffle_name,mlen):
    data_s_rmse = np.zeros((fnum,mlen,129))
    for i in range(fnum):
        if i%200 ==0:
            print(i)
        num = str(shuffle_name[i])
        filename = fname+num+'.wav'
        y, sr = librosa.load(filename)
        D = np.abs(librosa.stft(y))**2
        ss, phase = librosa.magphase(librosa.stft(y))
        rmse = librosa.feature.rmse(S=ss)
        rmse = rmse/np.max(rmse)
        rmse = np.transpose(rmse)
        S = librosa.feature.melspectrogram(S=D)
        S = np.transpose(S)
        if len(S)>=mlen:
            data_s_rmse[i][:,0]=rmse[-mlen:,0]
            data_s_rmse[i][:,1:]=S[-mlen:,:]
        else:
            data_s_rmse[i][-len(S):,0]=np.transpose(rmse)
            data_s_rmse[i][-len(S):,1:]=S
    return data_s_rmse

fem_speech = make_data('speech/FEMALE/',3552,x_fem,200)
mal_speech = make_data('speech/MALE/',3552,x_mal,200)

total_speech_train = np.concatenate([fem_speech[:3196],mal_speech[:3196]])
total_speech_test  = np.concatenate([fem_speech[3196:],mal_speech[3196:]])
total_speech = np.concatenate([total_speech_train,total_speech_test])
```

## 2. Speech-only analysis with Librosa and Keras

Although people these days seem to migrate to TF2.0 and PyTorch, I still use [original Keras](https://keras.io/) for its conciseness. Hope this code be fit to whatever flatform the reader uses.

```python
import tensorflow as tf
from keras.backend.tensorflow_backend import set_session
config = tf.ConfigProto()
config.gpu_options.per_process_gpu_memory_fraction = 0.2
set_session(tf.Session(config=config))

from keras.models import Sequential, Model
from keras.layers import TimeDistributed, Bidirectional, Concatenate
from keras.layers import Input, Embedding, LSTM, GRU, SimpleRNN, Lambda
from keras.layers.core import Dense, Dropout, Activation, Flatten, Reshape
from keras.layers.normalization import BatchNormalization
from keras.preprocessing import sequence
import keras.backend as K
from keras.callbacks import ModelCheckpoint
import keras.layers as layers

from keras import optimizers
adam_half = optimizers.Adam(lr=0.0005)
```

```python
##### class_weights
from sklearn.utils import class_weight
class_weights = class_weight.compute_class_weight('balanced', np.unique(fem_label), fem_label)

##### f1 score ftn.
from keras.callbacks import Callback
from sklearn import metrics
from sklearn.metrics import confusion_matrix, f1_score, precision_score, recall_score

class Metricsf1macro(Callback):
    def on_train_begin(self, logs={}):
        self.val_f1s = []
        self.val_recalls = []
        self.val_precisions = []
        self.val_f1s_w = []
        self.val_recalls_w = []
        self.val_precisions_w = []
    def on_epoch_end(self, epoch, logs={}):
        val_predict = np.asarray(self.model.predict(self.validation_data[0]))
        val_predict = np.argmax(val_predict,axis=1)
        val_targ = self.validation_data[1]
        _val_f1 = metrics.f1_score(val_targ, val_predict, average="macro")
        _val_f1_w = metrics.f1_score(val_targ, val_predict, average="weighted")
        _val_recall = metrics.recall_score(val_targ, val_predict, average="macro")
        _val_recall_w = metrics.recall_score(val_targ, val_predict, average="weighted")
        _val_precision = metrics.precision_score(val_targ, val_predict, average="macro")
        _val_precision_w = metrics.precision_score(val_targ, val_predict, average="weighted")
        self.val_f1s.append(_val_f1)
        self.val_recalls.append(_val_recall)
        self.val_precisions.append(_val_precision)
        self.val_f1s_w.append(_val_f1_w)
        self.val_recalls_w.append(_val_recall_w)
        self.val_precisions_w.append(_val_precision_w)
        print("— val_f1: %f — val_precision: %f — val_recall: %f"%(_val_f1, _val_precision, _val_recall))
        print("— val_f1_w: %f — val_precision_w: %f — val_recall_w: %f"%(_val_f1_w, _val_precision_w, _val_recall_w))

metricsf1macro = Metricsf1macro()
```

```python
def validate_bilstm(result,y,hidden_lstm,hidden_dim,cw,val_sp,bat_size,filename):
    model = Sequential()
    model.add(Bidirectional(LSTM(hidden_lstm), input_shape=(len(result[0]), len(result[0][0]))))
    model.add(layers.Dense(hidden_dim, activation='relu'))
    model.add(layers.Dense(int(max(y)+1), activation='softmax'))
    model.summary()
    model.compile(optimizer=adam_half, loss="sparse_categorical_crossentropy", metrics=["accuracy"])
    filepath=filename+"-{epoch:02d}-{val_acc:.4f}.hdf5"
    checkpoint = ModelCheckpoint(filepath, monitor='val_acc', verbose=1, mode='max')
    callbacks_list = [metricsf1macro,checkpoint]
    model.fit(result,y,validation_split=val_sp,epochs=100,batch_size=bat_size,callbacks=callbacks_list,class_weight=cw)

validate_bilstm(total_speech,total_label,64,128,class_weights,0.1,16,'model_icassp/total_bilstm')
```

## 3. Self-attentive BiLSTM

Remember: mel spectrogram still has a plenty of prosody-semantic information! Thus, we decided to apply a self-attentive embedding which has been successfully used in text procesisng. Before making up the module, in terms of *pure Keras* where F1 measure is removed (well if recent version has one, that's nice!), we need another definition of f1 score since additional input source is introduced (zero vector for attention source initialization).

```python
class Metricsf1macro_2input(Callback):
 def on_train_begin(self, logs={}):
  self.val_f1s = []
  self.val_recalls = []
  self.val_precisions = []
  self.val_f1s_w = []
  self.val_recalls_w = []
  self.val_precisions_w = []
 def on_epoch_end(self, epoch, logs={}):
  if len(self.validation_data)>2:
   val_predict = np.asarray(self.model.predict([self.validation_data[0],self.validation_data[1]]))
   val_predict = np.argmax(val_predict,axis=1)
   val_targ = self.validation_data[2]
  else:
   val_predict = np.asarray(self.model.predict(self.validation_data[0]))
   val_predict = np.argmax(val_predict,axis=1)
   val_targ = self.validation_data[1]
  _val_f1 = metrics.f1_score(val_targ, val_predict, average="macro")
  _val_f1_w = metrics.f1_score(val_targ, val_predict, average="weighted")
  _val_recall = metrics.recall_score(val_targ, val_predict, average="macro")
  _val_recall_w = metrics.recall_score(val_targ, val_predict, average="weighted")
  _val_precision = metrics.precision_score(val_targ, val_predict, average="macro")
  _val_precision_w = metrics.precision_score(val_targ, val_predict, average="weighted")
  self.val_f1s.append(_val_f1)
  self.val_recalls.append(_val_recall)
  self.val_precisions.append(_val_precision)
  self.val_f1s_w.append(_val_f1_w)
  self.val_recalls_w.append(_val_recall_w)
  self.val_precisions_w.append(_val_precision_w)
  print("— val_f1: %f — val_precision: %f — val_recall: %f"%(_val_f1, _val_precision, _val_recall))
  print("— val_f1_w: %f — val_precision_w: %f — val_recall_w: %f"%(_val_f1_w, _val_precision_w, _val_recall_w))

metricsf1macro_2input = Metricsf1macro_2input()
```

```python
def validate_rnn_self_drop(x_rnn,x_y,hidden_lstm,hidden_con,hidden_dim,cw,val_sp,bat_size,filename):
    char_r_input = Input(shape=(len(x_rnn[0]),len(x_rnn[0][0])),dtype='float32')
    r_seq = Bidirectional(LSTM(hidden_lstm,return_sequences=True))(char_r_input)
    r_att = Dense(hidden_con, activation='tanh')(r_seq)
    att_source   = np.zeros((len(x_rnn),hidden_con))
    att_test     = np.zeros((len(x_rnn),hidden_con))
    att_input    = Input(shape=(hidden_con,), dtype='float32')
    att_vec      = Dense(hidden_con,activation='relu')(att_input)
    att_vec      = Dropout(0.3)(att_vec)
    att_vec      = Dense(hidden_con,activation='relu')(att_vec)
    att_vec = Lambda(lambda x: K.batch_dot(*x, axes=(1,2)))([att_vec,r_att])
    att_vec = Dense(len(x_rnn[0]),activation='softmax')(att_vec)
    att_vec = layers.Reshape((len(x_rnn[0]),1))(att_vec)
    r_seq   = layers.multiply([att_vec,r_seq])
    r_seq   = Lambda(lambda x: K.sum(x, axis=1))(r_seq)
    r_seq   = Dense(hidden_dim, activation='relu')(r_seq)
    r_seq   = Dropout(0.3)(r_seq)
    r_seq   = Dense(hidden_dim, activation='relu')(r_seq)
    r_seq   = Dropout(0.3)(r_seq)
    main_output = Dense(int(max(x_y)+1),activation='softmax')(r_seq)
    model = Sequential()
    model = Model(inputs=[char_r_input,att_input],outputs=[main_output])
    model.summary()
    model.compile(optimizer=adam_half,loss="sparse_categorical_crossentropy",metrics=["accuracy"])
    filepath=filename+"-{epoch:02d}-{val_acc:.4f}.hdf5"
    checkpoint = ModelCheckpoint(filepath, monitor='val_acc', verbose=1, mode='max')
    callbacks_list = [metricsf1macro_2input,checkpoint]
    model.fit([x_rnn,att_source],x_y,validation_split=val_sp,epochs=100,batch_size= bat_size ,callbacks=callbacks_list,class_weight=cw)

validate_rnn_self_drop(total_speech,total_label,64,64,128,class_weights,0.1,16,'model_icassp/total_bilstm_att')
```

## 4. Parallel utilization of audio and text data

## 5. Multi-hop attention

## 6. Cross-attention
