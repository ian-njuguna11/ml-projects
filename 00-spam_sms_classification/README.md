```{.python .input  n=8}
!pip install statsmodels
```

```{.json .output n=8}
[
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "Collecting statsmodels\n  Downloading statsmodels-0.8.0-cp35-cp35m-manylinux1_x86_64.whl (6.2MB)\n\u001b[K    100% |\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588| 6.2MB 257kB/s ta 0:00:011\n\u001b[?25hRequirement already satisfied: scipy in /usr/local/lib/python3.5/dist-packages (from statsmodels)\nCollecting patsy (from statsmodels)\n  Downloading patsy-0.4.1-py2.py3-none-any.whl (233kB)\n\u001b[K    100% |\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588\u2588| 235kB 5.7MB/s eta 0:00:01\n\u001b[?25hRequirement already satisfied: pandas in /usr/local/lib/python3.5/dist-packages (from statsmodels)\nRequirement already satisfied: numpy>=1.8.2 in /usr/local/lib/python3.5/dist-packages (from scipy->statsmodels)\nRequirement already satisfied: six in /usr/local/lib/python3.5/dist-packages (from patsy->statsmodels)\nRequirement already satisfied: python-dateutil>=2 in /usr/local/lib/python3.5/dist-packages (from pandas->statsmodels)\nRequirement already satisfied: pytz>=2011k in /usr/local/lib/python3.5/dist-packages (from pandas->statsmodels)\nInstalling collected packages: patsy, statsmodels\nSuccessfully installed patsy-0.4.1 statsmodels-0.8.0\n"
 }
]
```

```{.python .input  n=1}
from IPython.display import IFrame
IFrame("report.pdf", width=1000, height=600)
```

```{.json .output n=1}
[
 {
  "data": {
   "text/html": "\n        <iframe\n            width=\"1000\"\n            height=\"600\"\n            src=\"report.pdf\"\n            frameborder=\"0\"\n            allowfullscreen\n        ></iframe>\n        ",
   "text/plain": "<IPython.lib.display.IFrame at 0x7f97e455f128>"
  },
  "execution_count": 1,
  "metadata": {},
  "output_type": "execute_result"
 }
]
```

```{.python .input  n=4}
# !head 'SMSSpam' | cut -d"$'\t'" -f2 | sed 's/,/ , /g' | tr ' ' '\n' | sort | uniq -c | sort -rn | less
```

# Required Libs and Class

+ We implemented two different classes such as
SMSBase, Util below cell.
+ Class SMSBase has common variables for different
classification algorithms and deep learning approaches. That class will be
extended by SMSClassification and SMSDL in the next cells.
+ Class Util has some
helper methods to plot and display data and result.

```{.python .input  n=9}
%matplotlib inline
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

plt.style.use('ggplot')

import spacy

import statsmodels.api as sm

from sklearn.pipeline import Pipeline

from sklearn.feature_extraction.text import CountVectorizer, TfidfTransformer
from sklearn.naive_bayes import MultinomialNB
from sklearn.svm import SVC
from sklearn.ensemble import RandomForestClassifier
from sklearn.decomposition import TruncatedSVD

from sklearn.model_selection import StratifiedKFold
from sklearn.model_selection import cross_val_score
from sklearn.model_selection import GridSearchCV
from sklearn.model_selection import learning_curve

from sklearn.metrics import confusion_matrix
from sklearn import metrics
from sklearn.metrics import classification_report

from sklearn.externals import joblib

import os.path

class SMSBase:
    # Spacy library is loading English dictionary.
    _nlp = spacy.load("en")
    
    def __init__(self, filename, frac=0.8):
        self._filename = filename
        self._features = ['class', 'context']
        
        self._df_raw = pd.read_csv(self._filename, sep='\t', names=self._features)
        self.__format_context()
        
        self.__extract_features()
        
        self._group_by_feature = self._df_raw .groupby('class')
        self._counts_by_features = self._group_by_feature.count().to_dict()['context']
        
        self.__split_test_train(frac)
        
    def __format_context(self):
        self._df_raw['context'] =  self._df_raw['context'].map(lambda text : text.rstrip())
        self._df_raw['context'] =  self._df_raw['context'].map(lambda text : text.replace(',', ' ,') if ',' in text else text)
    
    def __extract_features(self):
        self._df_raw['len']= self._df_raw['context'].map(lambda text : len(text))
        self._df_raw['n_words'] = self._df_raw['context'].map(lambda text : len(text.split(' ')))

        #updating features
        self._features = self._df_raw.columns
    
    def __split_test_train(self, frac):
        self._df_train = self._df_raw.sample(frac=frac)
        self._df_test = self._df_raw.drop(self._df_train.index)
    
    def describe(self):
        print('-' * 20 + 'Extended Dataset (Head)' + '-' * 20)
        display(self._df_raw.head())
        
        print('-' * 20 + 'Extended Dataset (Describe)' + '-' * 20)
        display(self._df_raw.describe())
        
        print('-' * 20 + 'Groupby Class (Describe)' + '-' * 20)
        display(self._group_by_feature.describe())
        
    def create_lemmas(self, c):
        tokens = self._nlp(c)
        return [token.lemma_ for token in tokens]
    
    def create_tokens(self, c):
        tokens = self._nlp(c)
        return [token for token in tokens]
    
    
class Util:
        
    def report_classification(model, df_train, df_test, X_features, y_feature):
        
        classes_train = np.unique(df_train[y_feature].values).tolist()
        classes_test = np.unique(df_test[y_feature].values).tolist()
        
        assert (classes_train == classes_test)
        
        classes = classes_train # The order of class is important!
        
        X_train = df_train[X_features].values.tolist()
        X_test = df_test[X_features].values.tolist()
        
        y_train = df_train[y_feature].values.tolist()
        y_test = df_test[y_feature].values.tolist()
        
        y_train_pred = model.predict(X_train)
        y_test_pred = model.predict(X_test)

        report_cm(y_train, y_test, y_train_pred, y_test_pred, classes)
        
    def report_cm(y_train, y_test, y_train_pred, y_test_pred, classes):
        figure, axes = plt.subplots(1, 2, figsize=(10,5))

        cm_test = confusion_matrix(y_test, y_test_pred)
        df_cm_test = pd.DataFrame(cm_test, index = classes, columns = classes)
        ax = sns.heatmap(df_cm_test, annot=True, ax = axes[0], square= True)
        ax.set_title('Test CM')

        cm_train = confusion_matrix(y_train, y_train_pred)
        df_cm_train = pd.DataFrame(cm_train, index = classes, columns = classes)
        ax = sns.heatmap(df_cm_train, annot=True, ax = axes[1], square= True)
        ax.set_title('Train CM')

        print('-' * 20 + 'Testing Performance' + '-' * 20)
        print(classification_report(y_test, y_test_pred, target_names = classes))
        print('acc: ', metrics.accuracy_score(y_test, y_test_pred))

        print('-' * 20 + 'Training Performance' + '-' * 20)
        print(classification_report(y_train, y_train_pred, target_names = classes))
        print('acc: ', metrics.accuracy_score(y_train, y_train_pred))
        
    
    def plot_cdf(p, 
             ax, 
             deltax=None, 
             xlog=False, 
             xlim=[0, 1], 
             deltay=0.25, 
             ylog=False, 
             ylim=[0,1], 
             xlabel = 'x'):

        df = pd.DataFrame(p, columns=[xlabel])
        display(df.describe())
        
        ecdf = sm.distributions.ECDF(p)
        x = ecdf.x
        y = ecdf.y
        assert len(x) == len(y)
        if deltax is not None:
            x_ticks = np.arange(xlim[0], xlim[1] + deltax, deltax)
            ax.set_xticks(x_ticks)

        ax.set_xlabel(xlabel)
        ax.set_xlim(xlim[0], xlim[1])
        ax.vlines(np.mean(p), min(y), max(y), color='red', label='mean', linewidth=2)
        ax.vlines(np.median(p), min(y), max(y), color='orange', label='median', linewidth=2)
        ax.vlines(np.mean(p) + 2 * np.std(p), min(y), max(y), color='blue', label='mean + 2 * std', linewidth=2)
        ax.vlines(np.mean(p) + 3 * np.std(p), min(y), max(y), color='green', label='mean + 3 * std', linewidth=2)

        y_ticks = np.arange(ylim[0], ylim[1] + deltay, deltay)
        ax.set_ylabel('CDF')
        ax.set_yticks(y_ticks)
        ax.set_ylim(ylim[0], ylim[1])

        if xlog is True:
            ax.set_xscale('log')

        if ylog is True:
            ax.set_yscale('log')


        ax.grid(which='minor', alpha=0.5)
        ax.grid(which='major', alpha=0.9)

        ax.legend(loc=4)

        sns.set_style('whitegrid')
        sns.regplot(x=x, y=y, fit_reg=False, scatter=True, ax = ax)
    
        
    def plot_class_dist(df, by):
        
        x_features = df.columns.drop(by)
        assert 0 < len(x_features)
        
        x_features = x_features[0]
        dist = df.groupby(by)[x_features].size() / len(df)
        display(dist)        
        sns.barplot(x=dist.index, y=dist.values)
        
    def plot_boxplot(df, by, y, ax):
        ax = sns.boxplot(x=by, y=y, data=df[[by,  y]], ax = ax)
        ax.set_yscale('log')
        
    def dump_pickle(obj,filename):
        joblib.dump(obj, filename)
        
    def load_pickle(filename):
        return joblib.load(filename)
```

```{.json .output n=9}
[
 {
  "name": "stderr",
  "output_type": "stream",
  "text": "/usr/local/lib/python3.5/dist-packages/statsmodels/compat/pandas.py:56: FutureWarning: The pandas.core.datetools module is deprecated and will be removed in a future version. Please use the pandas.tseries module instead.\n  from pandas.core import datetools\n"
 }
]
```

# Introduction

In this assessment, we classify SMS corpus. SMS corpus consists
of two columns such as label, context (message). Since data already has a target
(labelled)  feature we can apply supervised machine learning algorithms. Before
applying ML, we need to follow up those steps in the flowchart during this
experiments.
<img src="ml_steps.png">

After these operations, we applied
classification algorithms and deep learning. In the first part, you can find
classification's experiment. In the next second part, you can find deep learning
approach

## Part#1 - Classification
+ In this experiment, we used pipeline
approaching to apply a bunch of different operations respectively on the data.
The class of SMSClassfication has 3 different pipelines corresponding to
different ML algorithms, such as;
    + Navie Bayes (NB), 
    + SVM, 
    +
Random Forest Tree (RFT).
+ Before applying ML algorithms in each pipeline, we
used TF-IDF to exract features over SMS corpus.
+ In SVM and Random Forest
pipelining except for Naive Bayes, we used SVD (TruncatedSVD) to reduce of
dimension (number of features) of data since we have limited computational
resources (my personal laptop). So, we just focused on 50 features out of N. N
denotes the number of unique words that appear in the corpus. Before we are
creating the weights of the features on the bag-of-words by using TF-IDF,
CountVectorizer is creating token counts as a sparse matrix.
+ In training step,
we used GridSearchCV to find out the optimum hyperparameters. We used 10 fold
cross validation on StratifiedKFold since it guarantees that the weight of class
would be same with the original data in the each fold.

<img src="pipeline.png">

```{.python .input  n=2}
class SMSClassification(SMSBase):
    __pipelines = {}
    __params = {}
    __format_model_file_name = '{}_model.pkl'

    def __init__(self, filename, frac=0.8):
        super().__init__(filename, frac)
        
        self.__bow = CountVectorizer(analyzer=self.create_lemmas)
        self.__tfidf = TfidfTransformer()
        
        self.__svd = TruncatedSVD(n_components=50)

        self.__cv = StratifiedKFold(n_splits=10)
        
        self.__default_params = {
            'tfidf__use_idf': (True, False),
            'bow__analyzer': (self.create_lemmas, self.create_tokens),
        }
        
        self.__X = self._df_train['context'].values.tolist()
        self.__y = self._df_train['class'].values.tolist()
        
   
    def __create_pipeline(self, option='NB'):
                        
        if (option in self.__pipelines) is False:
                        
            if option is 'NB':
                classifier = MultinomialNB()
                pipeline = Pipeline([
                    ('bow', self.__bow),
                    ('tfidf', self.__tfidf),
                    ('classifier', classifier),
                ])

            elif option is 'SVM':
                classifier = SVC()
                pipeline = Pipeline([
                    ('bow', self.__bow),
                    ('tfidf', self.__tfidf),
                    ('svd', self.__svd),
                    ('classifier', classifier),
                ])
                
            elif option is 'RFT':
                classifier = RandomForestClassifier()
                pipeline = Pipeline([
                    ('bow', self.__bow),
                    ('tfidf', self.__tfidf),
                    ('svd', self.__svd),
                    ('classifier', classifier),
                ])
                
            else:
                classifier = MultinomialNB()

            self.__pipelines[option] = pipeline
            
            return pipeline

        else:
            return self.__pipelines[option]
            
            
    def __create_grid_search_params(self, option='NB'):
        
        if (option in self.__params) is False:
            if option is 'SVM':
                params = [
                    {
                      'classifier__C': [1, 10, 100, 1000], 
                      'classifier__kernel': ['linear']
                    },
                    {
                      'classifier__C': [1, 10, 100, 1000], 
                      'classifier__gamma': [0.001, 0.0001], 
                      'classifier__kernel': ['rbf']
                    },
                ]

                # merging two list of paramaters on the same list.
#                 params = list(map(lambda m : {**m, **self.__default_params}, params))
            else:
                params = self.__default_params

            self.__params[option] = params
        else:
            params = self.__params[option]
            
        return params

        
        
    def validate(self, option='NB'):
        
        pipeline = self.__create_pipeline(option)
        if pipeline is not None:            
            scores = cross_val_score(pipeline, 
                                     self.__X, 
                                     self.__y, 
                                     scoring='accuracy', 
                                     cv=self.__cv, 
                                     verbose=1, 
                                     n_jobs=-1)

            print('scores={}\nmean={} std={}'.format(scores, scores.mean(), scores.std()))
        else:
            print ("pipeline does not exist!")

        
    def train(self, option='NB', dump=True):
        
        pipeline = self.__create_pipeline(option)
        if pipeline is not None:
            
            params = self.__create_grid_search_params(option)
            
            grid = GridSearchCV(
                pipeline, 
                params, 
                refit=True, 
                n_jobs=-1, 
                scoring='accuracy', 
                cv=self.__cv)

            model = grid.fit(self.__X, self.__y)
            
            display('(Grid Search) Best Parameters:', )
            display(pd.DataFrame([model.best_params_]))

            if dump:
                model_file_name = self.__format_model_file_name.format(option)
                Util.dump_pickle(model, model_file_name)
                
            return model
                
        else:
            print('pipeline does not exist!')
            return None

    
    def test(self, X=None, model=None, model_file=None):
        
        if X is None:
            X = self.__X
        
        if model is None and model_file is None:
            print('Please, use either model or model_file')
            return []
        
        if model_file is not None and os.path.isfile(model_file):
            model = Util.load_pickle(model_file)
            print('{} file was loaded'.format(model_file))
            return model.predict(X)
        
        if model is not None:
            return model.predict(X)
        else:
            return []
```

```{.json .output n=2}
[
 {
  "name": "stderr",
  "output_type": "stream",
  "text": "/usr/local/lib/python3.5/dist-packages/statsmodels/compat/pandas.py:56: FutureWarning: The pandas.core.datetools module is deprecated and will be removed in a future version. Please use the pandas.tseries module instead.\n  from pandas.core import datetools\n"
 }
]
```

## Data Exploring

```{.python .input  n=12}
sms = SMSClassification('SMSSpam')
```

+ After loading data, to plot some statistical graphs (CDF, boxplot), we
calculated lenght of each context and number of words, which appears in each
context.
+ We calculated the distribution of classes which appear in the corpus.

```{.python .input  n=13}
sms.describe()
```

```{.json .output n=13}
[
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "--------------------Extended Dataset (Head)--------------------\n"
 },
 {
  "data": {
   "text/html": "<div>\n<style>\n    .dataframe thead tr:only-child th {\n        text-align: right;\n    }\n\n    .dataframe thead th {\n        text-align: left;\n    }\n\n    .dataframe tbody tr th {\n        vertical-align: top;\n    }\n</style>\n<table border=\"1\" class=\"dataframe\">\n  <thead>\n    <tr style=\"text-align: right;\">\n      <th></th>\n      <th>class</th>\n      <th>context</th>\n      <th>len</th>\n      <th>n_words</th>\n    </tr>\n  </thead>\n  <tbody>\n    <tr>\n      <th>0</th>\n      <td>ham</td>\n      <td>Go until jurong point , crazy.. Available only...</td>\n      <td>112</td>\n      <td>21</td>\n    </tr>\n    <tr>\n      <th>1</th>\n      <td>ham</td>\n      <td>Ok lar... Joking wif u oni...</td>\n      <td>29</td>\n      <td>6</td>\n    </tr>\n    <tr>\n      <th>2</th>\n      <td>spam</td>\n      <td>Free entry in 2 a wkly comp to win FA Cup fina...</td>\n      <td>155</td>\n      <td>28</td>\n    </tr>\n    <tr>\n      <th>3</th>\n      <td>ham</td>\n      <td>U dun say so early hor... U c already then say...</td>\n      <td>49</td>\n      <td>11</td>\n    </tr>\n    <tr>\n      <th>4</th>\n      <td>ham</td>\n      <td>Nah I don't think he goes to usf , he lives ar...</td>\n      <td>62</td>\n      <td>14</td>\n    </tr>\n  </tbody>\n</table>\n</div>",
   "text/plain": "  class                                            context  len  n_words\n0   ham  Go until jurong point , crazy.. Available only...  112       21\n1   ham                      Ok lar... Joking wif u oni...   29        6\n2  spam  Free entry in 2 a wkly comp to win FA Cup fina...  155       28\n3   ham  U dun say so early hor... U c already then say...   49       11\n4   ham  Nah I don't think he goes to usf , he lives ar...   62       14"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "--------------------Extended Dataset (Describe)--------------------\n"
 },
 {
  "data": {
   "text/html": "<div>\n<style>\n    .dataframe thead tr:only-child th {\n        text-align: right;\n    }\n\n    .dataframe thead th {\n        text-align: left;\n    }\n\n    .dataframe tbody tr th {\n        vertical-align: top;\n    }\n</style>\n<table border=\"1\" class=\"dataframe\">\n  <thead>\n    <tr style=\"text-align: right;\">\n      <th></th>\n      <th>len</th>\n      <th>n_words</th>\n    </tr>\n  </thead>\n  <tbody>\n    <tr>\n      <th>count</th>\n      <td>5572.000000</td>\n      <td>5572.000000</td>\n    </tr>\n    <tr>\n      <th>mean</th>\n      <td>80.811917</td>\n      <td>16.031407</td>\n    </tr>\n    <tr>\n      <th>std</th>\n      <td>60.273482</td>\n      <td>11.832019</td>\n    </tr>\n    <tr>\n      <th>min</th>\n      <td>2.000000</td>\n      <td>1.000000</td>\n    </tr>\n    <tr>\n      <th>25%</th>\n      <td>36.000000</td>\n      <td>7.000000</td>\n    </tr>\n    <tr>\n      <th>50%</th>\n      <td>62.000000</td>\n      <td>13.000000</td>\n    </tr>\n    <tr>\n      <th>75%</th>\n      <td>123.000000</td>\n      <td>24.000000</td>\n    </tr>\n    <tr>\n      <th>max</th>\n      <td>910.000000</td>\n      <td>175.000000</td>\n    </tr>\n  </tbody>\n</table>\n</div>",
   "text/plain": "               len      n_words\ncount  5572.000000  5572.000000\nmean     80.811917    16.031407\nstd      60.273482    11.832019\nmin       2.000000     1.000000\n25%      36.000000     7.000000\n50%      62.000000    13.000000\n75%     123.000000    24.000000\nmax     910.000000   175.000000"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "--------------------Groupby Class (Describe)--------------------\n"
 },
 {
  "data": {
   "text/html": "<div>\n<style>\n    .dataframe thead tr:only-child th {\n        text-align: right;\n    }\n\n    .dataframe thead th {\n        text-align: left;\n    }\n\n    .dataframe tbody tr th {\n        vertical-align: top;\n    }\n</style>\n<table border=\"1\" class=\"dataframe\">\n  <thead>\n    <tr>\n      <th></th>\n      <th colspan=\"8\" halign=\"left\">len</th>\n      <th colspan=\"8\" halign=\"left\">n_words</th>\n    </tr>\n    <tr>\n      <th></th>\n      <th>count</th>\n      <th>mean</th>\n      <th>std</th>\n      <th>min</th>\n      <th>25%</th>\n      <th>50%</th>\n      <th>75%</th>\n      <th>max</th>\n      <th>count</th>\n      <th>mean</th>\n      <th>std</th>\n      <th>min</th>\n      <th>25%</th>\n      <th>50%</th>\n      <th>75%</th>\n      <th>max</th>\n    </tr>\n    <tr>\n      <th>class</th>\n      <th></th>\n      <th></th>\n      <th></th>\n      <th></th>\n      <th></th>\n      <th></th>\n      <th></th>\n      <th></th>\n      <th></th>\n      <th></th>\n      <th></th>\n      <th></th>\n      <th></th>\n      <th></th>\n      <th></th>\n      <th></th>\n    </tr>\n  </thead>\n  <tbody>\n    <tr>\n      <th>ham</th>\n      <td>4825.0</td>\n      <td>71.775337</td>\n      <td>58.783939</td>\n      <td>2.0</td>\n      <td>33.0</td>\n      <td>52.0</td>\n      <td>93.0</td>\n      <td>910.0</td>\n      <td>4825.0</td>\n      <td>14.726010</td>\n      <td>11.974647</td>\n      <td>1.0</td>\n      <td>7.0</td>\n      <td>11.0</td>\n      <td>19.0</td>\n      <td>175.0</td>\n    </tr>\n    <tr>\n      <th>spam</th>\n      <td>747.0</td>\n      <td>139.180723</td>\n      <td>29.067007</td>\n      <td>13.0</td>\n      <td>133.0</td>\n      <td>149.0</td>\n      <td>158.0</td>\n      <td>224.0</td>\n      <td>747.0</td>\n      <td>24.463186</td>\n      <td>6.001311</td>\n      <td>2.0</td>\n      <td>22.0</td>\n      <td>26.0</td>\n      <td>28.0</td>\n      <td>37.0</td>\n    </tr>\n  </tbody>\n</table>\n</div>",
   "text/plain": "          len                                                           \\\n        count        mean        std   min    25%    50%    75%    max   \nclass                                                                    \nham    4825.0   71.775337  58.783939   2.0   33.0   52.0   93.0  910.0   \nspam    747.0  139.180723  29.067007  13.0  133.0  149.0  158.0  224.0   \n\n      n_words                                                      \n        count       mean        std  min   25%   50%   75%    max  \nclass                                                              \nham    4825.0  14.726010  11.974647  1.0   7.0  11.0  19.0  175.0  \nspam    747.0  24.463186   6.001311  2.0  22.0  26.0  28.0   37.0  "
  },
  "metadata": {},
  "output_type": "display_data"
 }
]
```

```{.python .input  n=27}
Util.plot_class_dist(sms._df_raw, 'class')
```

```{.json .output n=27}
[
 {
  "data": {
   "text/plain": "class\nham     0.865937\nspam    0.134063\nName: context, dtype: float64"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "data": {
   "image/png": "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\n",
   "text/plain": "<matplotlib.figure.Figure at 0x7f753d86add8>"
  },
  "metadata": {},
  "output_type": "display_data"
 }
]
```

+ According to the next graph CDF of number of words in context, 
    + 50% of
the corpus consists of contexts which have less than 13 words.
    + 90% of the
corpus consists of contexts which have less than roughly 32 words.
+ According
to the next graph boxplot of number of words in context, 
    + The classes
(ham, spam) have uniquely identified distribution, clearly.

```{.python .input  n=28}
n_words_in_context = sms._df_raw['n_words'].values.tolist()

figure, axes = plt.subplots(1, 2, figsize=(15,5))
Util.plot_cdf(n_words_in_context, 
         axes[0], 
         xlim=[0, np.mean(n_words_in_context) + 3 * np.std(n_words_in_context) + 50],
         deltay = 0.05,
         ylim=[0, 1.00], xlabel='number of words')

Util.plot_boxplot(sms._df_raw, 'class', 'n_words', axes[1])
```

```{.json .output n=28}
[
 {
  "data": {
   "text/html": "<div>\n<style>\n    .dataframe thead tr:only-child th {\n        text-align: right;\n    }\n\n    .dataframe thead th {\n        text-align: left;\n    }\n\n    .dataframe tbody tr th {\n        vertical-align: top;\n    }\n</style>\n<table border=\"1\" class=\"dataframe\">\n  <thead>\n    <tr style=\"text-align: right;\">\n      <th></th>\n      <th>number of words</th>\n    </tr>\n  </thead>\n  <tbody>\n    <tr>\n      <th>count</th>\n      <td>5572.000000</td>\n    </tr>\n    <tr>\n      <th>mean</th>\n      <td>16.031407</td>\n    </tr>\n    <tr>\n      <th>std</th>\n      <td>11.832019</td>\n    </tr>\n    <tr>\n      <th>min</th>\n      <td>1.000000</td>\n    </tr>\n    <tr>\n      <th>25%</th>\n      <td>7.000000</td>\n    </tr>\n    <tr>\n      <th>50%</th>\n      <td>13.000000</td>\n    </tr>\n    <tr>\n      <th>75%</th>\n      <td>24.000000</td>\n    </tr>\n    <tr>\n      <th>max</th>\n      <td>175.000000</td>\n    </tr>\n  </tbody>\n</table>\n</div>",
   "text/plain": "       number of words\ncount      5572.000000\nmean         16.031407\nstd          11.832019\nmin           1.000000\n25%           7.000000\n50%          13.000000\n75%          24.000000\nmax         175.000000"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "data": {
   "image/png": "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\nl1u8ZevG/aZas6+7jctoNLbre+rMOluu7BlrZ8tVS9Uuqb3nYzaVK7PZzAMPPICXlxceHh64ubnZ\nXjs5OdGrVy8ee+wxXn/99Qb9XnvtNUaOHMmECRP4y1/+woEDB/Dy8sLBwcG2LycnJywWS4N9v/fe\newBMnjyZ7du33xbPlStXuHbtGsHBwXh5eREeHs6VK1cYPHhwi97rRx99xJdffsmnn37KiRMniImJ\nsb1/V1fXBrmwWq235aWrHleNqaqqYuvWrQQHB9vW3elyibq6OhwcHLBYLBQXF9sxavvbvXs3x44d\ns3cYtyksLARg4cKFdo6kcUOGDGH8+PH2DkNE2lGbF34Wi4WioiLbcmFhIRaLpdG2goICPD098fT0\n5NFHHwWuX1/i4eFBQUEBvXv3bnJfubm59/4NPPrDthn3Fn2//m9L9tWabRvj5eXVLu+pc7ue5fr6\n+k6VK3vGquOq5ZrKVUlJCRUVFeTm5lJUVERlZWWD17179+bo0aNkZWXh6OhIfHw8P/vZz8jNzeWB\nBx4gJyeHvXv3UldXR25uboNjuKqqisLCwkb3XVNT0+j69PR01q5dy4YNGwA4ceIEP/xhy/4unjt3\njqysLEaNGsX06dOJiooiNzfXFpvFYiEhIYGnnnqKlJSURmNoLFd9+/alK3JwcCA6OppPP/3Utu5O\nl0s4OjpSU1PDtWvXbJdLNKWr5gzAzc0Nk8lk7zBu4+TkBNAhY4PreevKx4WI3K7NC7/Q0FB2797N\nqFGjyMjIwGKx2K4P6d27N5WVleTl5dGjRw+OHz/O7NmzOXToEIWFhYwZM4aioiKKi4vx9PRs61D/\nxWAAh+4wJALT1Dntt18RkWZ4eXnx7LPPMmfOHNvNXRwdHfnxj3/M+vXr6dOnDz/5yU9Yu3YtX3zx\nRYvHbezGLgABAQF8//vf55VXXgFg2LBhLb5G0Nvbmy1btvCnP/0Jo9HIT3/6UwAGDBjAjBkziIuL\n489//jNz5syhf//+LSpgujKTyXRbkXCnyyUef/xxtmzZQn19PRMnTmx27MuXWzPPpXN56qmneOqp\np+wdRqfUlY8LkftVUz/oGKxNzb28R3bs2EFaWhoGg4HIyEgyMzNxcXEhPDyc1NRUduzYAcAjjzzC\nmDFjqKysZN26dVRUVFBbW8vYsWMb3DDgTpr7A1a3ci5cyGh2HNPmPS17Y/dQ38TrN5C4/IPmb/zQ\n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\n",
   "text/plain": "<matplotlib.figure.Figure at 0x7f753d8b4e48>"
  },
  "metadata": {},
  "output_type": "display_data"
 }
]
```

+ According to the next graph CDF of length of words in context, 
    + 50% of
the corpus consists of contexts whose length are less than roughly 62.
    + 90%
of the corpus consists of contexts whose length are less than roughly 155.
+
According to the next graph boxplot of number of words in context, 
    + The
classes (ham, spam) have uniquely identified distribution, clearly.

```{.python .input  n=29}
len_of_context = sms._df_raw['len'].values.tolist()

figure, axes = plt.subplots(1, 2, figsize=(15,5))

Util.plot_cdf(len_of_context, 
         axes[0], 
         xlim=[0, np.mean(len_of_context) + 3 * np.std(len_of_context) + 50],
         deltay = 0.05,
         ylim=[0, 1.00], xlabel='len of context')

Util.plot_boxplot(sms._df_raw, 'class', 'len', axes[1])
```

```{.json .output n=29}
[
 {
  "data": {
   "text/html": "<div>\n<style>\n    .dataframe thead tr:only-child th {\n        text-align: right;\n    }\n\n    .dataframe thead th {\n        text-align: left;\n    }\n\n    .dataframe tbody tr th {\n        vertical-align: top;\n    }\n</style>\n<table border=\"1\" class=\"dataframe\">\n  <thead>\n    <tr style=\"text-align: right;\">\n      <th></th>\n      <th>len of context</th>\n    </tr>\n  </thead>\n  <tbody>\n    <tr>\n      <th>count</th>\n      <td>5572.000000</td>\n    </tr>\n    <tr>\n      <th>mean</th>\n      <td>80.811917</td>\n    </tr>\n    <tr>\n      <th>std</th>\n      <td>60.273482</td>\n    </tr>\n    <tr>\n      <th>min</th>\n      <td>2.000000</td>\n    </tr>\n    <tr>\n      <th>25%</th>\n      <td>36.000000</td>\n    </tr>\n    <tr>\n      <th>50%</th>\n      <td>62.000000</td>\n    </tr>\n    <tr>\n      <th>75%</th>\n      <td>123.000000</td>\n    </tr>\n    <tr>\n      <th>max</th>\n      <td>910.000000</td>\n    </tr>\n  </tbody>\n</table>\n</div>",
   "text/plain": "       len of context\ncount     5572.000000\nmean        80.811917\nstd         60.273482\nmin          2.000000\n25%         36.000000\n50%         62.000000\n75%        123.000000\nmax        910.000000"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "data": {
   "image/png": "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\n",
   "text/plain": "<matplotlib.figure.Figure at 0x7f753e49c320>"
  },
  "metadata": {},
  "output_type": "display_data"
 }
]
```

+ Filtering context by minimum number of words == 1

```{.python .input  n=30}
sms._df_raw[sms._df_raw['n_words'] == 1]['context'].values.tolist()
```

```{.json .output n=30}
[
 {
  "data": {
   "text/plain": "['Yup',\n 'Thanx...',\n 'Okie...',\n 'Ok..',\n 'Beerage?',\n 'Ok...',\n 'Ok...',\n 'Ok...',\n 'Ok...',\n '645',\n 'Ok...',\n 'Ok',\n 'Ok.',\n 'Ok...',\n 'Okie...',\n 'Yup...',\n 'Ok...',\n 'Okie',\n 'Ok...',\n 'Okie',\n 'ALRITE',\n 'Anything...',\n 'Ok',\n 'staff.science.nus.edu.sg/~phyhcmk/teaching/pc1323',\n 'Ok...',\n ':)',\n 'Ok.',\n 'Nite...',\n 'Ok.',\n 'Okie',\n 'Ok.',\n 'Okie...',\n 'G.W.R',\n 'Ok',\n 'Havent.',\n 'Anytime...',\n 'Ok...',\n 'Okie',\n 'Ok',\n 'Yup']"
  },
  "execution_count": 30,
  "metadata": {},
  "output_type": "execute_result"
 }
]
```

+ Filtering context by mean number of words == 16

```{.python .input  n=31}
sms._df_raw[sms._df_raw['n_words'] == 16]['context'].values.tolist()
```

```{.json .output n=31}
[
 {
  "data": {
   "text/plain": "['Even my brother is not like to speak with me. They treat me like aids patent.',\n \"I'm back &amp; we're packing the car now , I'll let you know if there's room\",\n \"K fyi x has a ride early tomorrow morning but he's crashing at our place tonight\",\n \"Its not the same here. Still looking for a job. How much do Ta's earn there.\",\n \"I'm still looking for a car to buy. And have not gone 4the driving test yet.\",\n 'Text her. If she doesnt reply let me know so i can have her log in',\n 'Busy here. Trying to finish for new year. I am looking forward to finally meeting you...',\n 'Here is your discount code RP176781. To stop further messages reply stop. www.regalportfolio.co.uk. Customer Services 08717205546',\n 'Hey so this sat are we going for the intro pilates only? Or the kickboxing too?',\n 'From here after The performance award is calculated every two month.not for current one month period..',\n 'Hmmm...k...but i want to change the field quickly da:-)i wanna get system administrator or network administrator..',\n 'Yep , the great loxahatchee xmas tree burning of  &lt;#&gt;  starts in an hour',\n \"Alright , I'll head out in a few minutes , text me where to meet you\",\n '7 at esplanade.. Do \u00fc mind giving me a lift cos i got no car today..',\n 'Hi hope u get this txt~journey hasnt been gd ,now about 50 mins late I think.',\n 'Yes..he is really great..bhaji told kallis best cricketer after sachin in world:).very tough to get out.',\n \"They don't put that stuff on the roads to keep it from getting slippery over there?\",\n 'Tell them u have a headache and just want to use 1 hour of sick time.',\n 'Mind blastin.. No more Tsunamis will occur from now on.. Rajnikant stopped swimming in Indian Ocean..:-D',\n \"MAKE SURE ALEX KNOWS HIS BIRTHDAY IS OVER IN FIFTEEN MINUTES AS FAR AS YOU'RE CONCERNED\",\n \"Well imma definitely need to restock before thanksgiving , I'll let you know when I'm out\",\n 'My sister in law , hope you are having a great month. Just saying hey. Abiola',\n 'Will purchase d stuff today and mail to you. Do you have a po box number?',\n 'meet you in corporation st outside gap \u2026 you can see how my mind is working!',\n \"There generally isn't one. It's an uncountable noun - u in the dictionary. pieces of research?\",\n 'I love to give massages. I use lots of baby oil... What is your fave position?',\n 'Yoyyooo u know how to change permissions for a drive in mac. My usb flash drive',\n \"Doesn't g have class early tomorrow and thus shouldn't be trying to smoke at  &lt;#&gt;\",\n 'I.ll give her once i have it. Plus she said grinule greet you whenever we speak',\n \"What time you think you'll have it? Need to know when I should be near campus\",\n 'Yo chad which gymnastics class do you wanna take? The site says Christians class is full..',\n 'If he started searching he will get job in few days.he have great potential and talent.',\n 'Up to \u00fc... \u00dc wan come then come lor... But i din c any stripes skirt...',\n 'Yes..he is really great..bhaji told kallis best cricketer after sachin in world:).very tough to get out.',\n \"Tell me they're female :V how're you throwing in? We're deciding what all to get now\",\n 'Meeting u is my work. . . Tel me when shall i do my work tomorrow',\n \"No I'm good for the movie , is it ok if I leave in an hourish?\",\n 'Not tonight mate. Catching up on some sleep. This is my new number by the way.',\n 'Damn , can you make it tonight or do you want to just wait til tomorrow',\n 'I want to send something that can sell fast.  &lt;#&gt; k is not easy money.',\n 'U ned to convince him tht its not possible witot hurting his feeling its the main',\n 'Wat so late still early mah. Or we juz go 4 dinner lor. Aiya i dunno...',\n 'In fact when do you leave? I think addie goes back to school tues or wed',\n \"Nice. Wait...should you be texting right now? I'm not gonna pay your ticket , ya know!\",\n 'He says hi and to get your ass back to south tampa (preferably at a kegger)',\n 'Free Top ringtone -sub to weekly ringtone-get 1st week free-send SUBPOLY to 81618-?3 per week-stop sms-08718727870',\n 'Yes! I am a one woman man! Please tell me your likes and dislikes in bed...',\n 'Then u better go sleep.. Dun disturb u liao.. U wake up then msg me lor..',\n 'Now , whats your house # again ? And do you have any beer there ?',\n \"Yeah. I got a list with only u and Joanna if I'm feeling really anti social\",\n 'You got job in wipro:)you will get every thing in life in 2 or 3 years.',\n 'Today my system sh get ready.all is well and i am also in the deep well',\n 'Dont know supports ass and srt i thnk. I think ps3 can play through usb too',\n 'And stop being an old man. You get to build snowman snow angels and snowball fights.',\n \"I'll be in sch fr 4-6... I dun haf da book in sch... It's at home...\",\n \"Sorry , I'll call later in meeting any thing related to trade please call Arul. &lt;#&gt;\",\n 'you are sweet as well , princess. Please tell me your likes and dislikes in bed...',\n 'If he started searching he will get job in few days.he have great potential and talent.',\n 'I have 2 sleeping bags , 1 blanket and paper and  phone details. Anything else?',\n 'tddnewsletter@emc1.co.uk (More games from TheDailyDraw) Dear Helen , Dozens of Free Games - with great prizesWith..',\n 'Hmm , too many of them unfortunately... Pics obviously arent hot cakes. Its kinda fun tho',\n 'Is there coming friday is leave for pongal?do you get any news from your work place.',\n \"Lol that's different. I don't go trying to find every real life photo you ever took.\",\n \"Dun b sad.. It's over.. Dun thk abt it already. Concentrate on ur other papers k.\",\n \"Can you just come in for a sec? There's somebody here I want you to see\",\n 'Got fujitsu , ibm , hp , toshiba... Got a lot of model how to say...',\n 'Sorry da:)i was thought of calling you lot of times:)lil busy.i will call you at noon..',\n 'There r many model..sony ericson also der.. &lt;#&gt; ..it luks good bt i forgot modl no',\n \"Nope but i'm going home now then go pump petrol lor... Like going 2 rain soon...\",\n 'Hope you enjoyed your new content. text stop to 61610 to unsubscribe. help:08712400602450p Provided by tones2you.co.uk',\n \"It's justbeen overa week since we broke up and already our brains are going to mush!\",\n 'Long after I quit. I get on only like 5 minutes a day as it is.',\n 'Forgot it takes me 3 years to shower , sorry. Where you at/your phone dead yet?',\n \"She's fine. Good to hear from you. How are you my dear? Happy new year oh.\",\n 'I want to be there so i can kiss you and feel you next to me',\n 'He says hi and to get your ass back to south tampa (preferably at a kegger)',\n \"Babe? You said 2 hours and it's been almost 4 ... Is your internet down ?\",\n 'Helloooo... Wake up..! \"Sweet\" \"morning\" \"welcomes\" \"You\" \"Enjoy\" \"This Day\" \"with full of joy\".. \"GUD MRNG\".',\n 'Nothing lor... A bit bored too... Then y dun u go home early 2 sleep today...',\n 'Sary just need Tim in the bollox &it hurt him a lot so he tol me!',\n \"That's y i said it's bad dat all e gals know u... Wat u doing now?\",\n \"Mystery solved! Just opened my email and he's sent me another batch! Isn't he a sweetie\",\n 'I will lick up every drop :) are you ready to use your mouth as well?',\n \"Yup i'm still having coffee wif my frens... My fren drove she'll give me a lift...\",\n 'I will send them to your email. Do you mind  &lt;#&gt;  times per night?',\n \"Lol your right. What diet? Everyday I cheat anyway. I'm meant to be a fatty :(\",\n 'I know girls always safe and selfish know i got it pa. Thank you. good night.',\n 'Then \u00fc ask dad to pick \u00fc up lar... \u00dc wan 2 stay until 6 meh...',\n \"You said to me before i went back to bed that you can't sleep for anything.\",\n 'No it will reach by 9 only. She telling she will be there. I dont know',\n 'Hey... are you going to quit soon? Xuhui and i working till end of the month',\n \"I'll get there at 3 , unless you guys want me to come some time sooner\",\n 'Dad wanted to talk about the apartment so I got a late start , omw now',\n 'Meeting u is my work. . . Tel me when shall i do my work tomorrow',\n 'Free Top ringtone -sub to weekly ringtone-get 1st week free-send SUBPOLY to 81618-?3 per week-stop sms-08718727870',\n 'Are you sure you don\\'t mean \"get here , we made you hold all the weed\"',\n 'Short But Cute: \"Be a good person , but dont try to prove it..\" .Gud noon....',\n 'I got arrested for possession at , I shit you not ,  &lt;TIME&gt;  pm',\n 'Went to pay rent. So i had to go to the bank to authorise the payment.',\n 'Yes ammae....life takes lot of turns you can only sit and try to hold the steering...',\n 'Happy or sad  , one thing about past is- \"Its no more\" GOOD MORNING :-):-).',\n 'Short But Cute: \"Be a good person , but dont try to prove it..\" .Gud noon....',\n \"Sounds like a plan! Cardiff is still here and still cold! I'm sitting on the radiator!\",\n 'Haha... dont be angry with yourself... Take it as a practice for the real thing. =)',\n 'I am taking you for italian food. How about a pretty dress with no panties? :)',\n 'Dont flatter yourself... Tell that man of mine two pints of carlin in ten minutes please....',\n 'Lol or I could just starve and lose a pound by the end of the day.',\n 'dont make ne plans for nxt wknd coz she wants us to come down then ok',\n 'Loans for any purpose even if you have Bad Credit! Tenants Welcome. Call NoWorriesLoans.com on 08717111821',\n \"Quite ok but a bit ex... U better go eat smth now else i'll feel guilty...\",\n \"I ain't answerin no phone at what is actually a pretty reasonable hour but I'm sleepy\",\n 'This single single answers are we fighting? Plus i said am broke and you didnt reply',\n \"He's in lag. That's just the sad part but we keep in touch thanks to skype\",\n 'Well the general price is  &lt;#&gt; /oz , let me know if/when/how much you want',\n 'No da:)he is stupid da..always sending like this:)don believe any of those message.pandy is a mental:)',\n \"Aight , I'm chillin in a friend's room so text me when you're on the way\",\n 'Hey mr whats the name of that bill brison book the one about language and words',\n 'Are there TA jobs available? Let me know please cos i really need to start working',\n \"Is ur changes 2 da report big? Cos i've already made changes 2 da previous report.\",\n \"K , jason says he's gonna be around so I'll be up there around  &lt;#&gt;\",\n \"Fyi I'm taking a quick shower , be at epsilon in like  &lt;#&gt;  min\",\n 'Only just got this message , not ignoring you. Yes , i was. Shopping that is',\n \"Yeah I should be able to , I'll text you when I'm ready to meet up\",\n \"What do u reckon as need 2 arrange transport if u can't do it , thanks\",\n \"Man this bus is so so so slow. I think you're gonna get there before me\",\n \"I'm stuck in da middle of da row on da right hand side of da lt...\",\n \"And I don't plan on staying the night but I prolly won't be back til late\",\n 'Ya i knw u vl giv..its ok thanks kano..anyway enjoy wit ur family wit 1st salary..:-);-)',\n \"Jason says it's cool if we pick some up from his place in like an hour\",\n 'I dont know ask to my brother. Nothing problem some thing that. Just i told .',\n 'Its worse if if uses half way then stops. Its better for him to complete it.',\n \"What i mean was i left too early to check , cos i'm working a 9-6.\",\n \"A few people are at the game , I'm at the mall with iouri and kaila\",\n 'Is there coming friday is leave for pongal?do you get any news from your work place.',\n 'How many times i told in the stage all use to laugh. You not listen aha.',\n \"That sucks. I'll go over so u can do my hair. You'll do it free right?\",\n 'No problem with the renewal. I.ll do it right away but i dont know his details.',\n 'I\u2018ve got some salt , you can rub it in my open wounds if you like!',\n 'Helloooo... Wake up..! \"Sweet\" \"morning\" \"welcomes\" \"You\" \"Enjoy\" \"This Day\" \"with full of joy\".. \"GUD MRNG\".',\n \"I can. But it will tell quite long , cos i haven't finish my film yet...\",\n \"Guai... \u00dc shd haf seen him when he's naughty... \u00dc so free today? Can go jogging...\",\n 'I can make it up there , squeezed  &lt;#&gt;  bucks out of my dad',\n \"So how are you really. What are you up to. How's the masters. And so on.\",\n \"And that's fine , I got enough bud to last most of the night at least\",\n 'For the most sparkling shopping breaks from 45 per person; call 0121 2025050 or visit www.shortbreaks.org.uk',\n 'Perhaps * is much easy give your account identification , so i will tomorrow at UNI',\n 'Yes. Please leave at  &lt;#&gt; . So that at  &lt;#&gt;  we can leave',\n 'Want explicit SEX in 30 secs? Ring 02073162414 now! Costs 20p/min Gsex POBOX 2667 WC1N 3XX']"
  },
  "execution_count": 31,
  "metadata": {},
  "output_type": "execute_result"
 }
]
```

+ Filtering context by length of words == 18

```{.python .input  n=33}
sms._df_raw[sms._df_raw['len'] == 18]['context'].values.tolist()
```

```{.json .output n=33}
[
 {
  "data": {
   "text/plain": "['Watching tv lor...',\n 'Was the farm open?',\n \"Annoying isn't it.\",\n 'Ok i juz receive..',\n 'Okay... We wait ah',\n 'At home by the way',\n 'Wat r u doing now?',\n \"Can... I'm free...\",\n 'Sorry. || mail? ||',\n 'Wat u doing there?',\n 'ringtoneking 84484',\n 'Wat time \u00fc finish?',\n 'Thinking of u ;) x',\n 'WHAT TIME U WRKIN?']"
  },
  "execution_count": 33,
  "metadata": {},
  "output_type": "execute_result"
 }
]
```

+ Filtering context by median length of words == 60

```{.python .input  n=34}
sms._df_raw[sms._df_raw['len'] == 60]['context'].values.tolist()
```

```{.json .output n=34}
[
 {
  "data": {
   "text/plain": "['Yeah sure , give me a couple minutes to track down my wallet',\n 'sorry , no , have got few things to do. may be in pub later.',\n 'Today am going to college so am not able to atten the class.',\n 'No no:)this is kallis home ground.amla home town is durban:)',\n 'I want to show you the world , princess :) how about europe?',\n 'Horrible gal. Me in sch doing some stuff. How come u got mc?',\n 'Eat jap done oso aft ur lect wat... \u00dc got lect at 12 rite...',\n 'Hai priya are you right. What doctor said pa. Where are you.',\n \"Yes , i'm small kid.. And boost is the secret of my energy..\",\n 'Must come later.. I normally bathe him in da afternoon mah..',\n 'Wish u many many returns of the day.. Happy birthday vikky..',\n 'Hiya , had a good day? Have you spoken to since the weekend?',\n 'Sorry da thangam , very very sorry i am held up with prasad.',\n 'I want to be there so i can kiss you and feel you next to me',\n 'Yar lor... Keep raining non stop... Or u wan 2 go elsewhere?',\n 'Thank u. IT BETTER WORK OUT CAUSE I WILL FEEL USED OTHERWISE',\n 'K... Must book a not huh? so going for yoga basic on sunday?',\n \"Yeah we wouldn't leave for an hour at least , how's 4 sound?\",\n 'That means you got an A in epi , she.s fine. She.s here now.',\n 'Sir , i am waiting for your call , once free please call me.',\n \"\u00dc neva tell me how i noe... I'm not at home in da aft wat...\",\n 'I will be gentle princess! We will make sweet gentle love...',\n 'His frens go then he in lor. Not alone wif my mum n sis lor.',\n 'Did you get any gift? This year i didnt get anything. So bad',\n \"Thought we could go out for dinner. I'll treat you! Seem ok?\",\n \"Actually i'm waiting for 2 weeks when they start putting ad.\",\n 'Yes. I come to nyc for audiitions and am trying to relocate.',\n 'Baaaaaaaabe! Wake up ! I miss you ! I crave you! I need you!',\n 'Ya but it cant display internal subs so i gotta extract them',\n \"Actually I decided I was too hungry so I haven't left yet :V\",\n 'I didnt get ur full msg..sometext is missing , send it again',\n 'Did you say bold , then torch later. Or one torch and 2bold?',\n \"I'm home , my love ... If your still awake ... *loving kiss*\",\n 'Uncle G , just checking up on you. Do have a rewarding month',\n 'Compliments to you. Was away from the system. How your side.']"
  },
  "execution_count": 34,
  "metadata": {},
  "output_type": "execute_result"
 }
]
```

## Validation Stage

+ In validation stage, we applied 3 different pipelining
without choosing sufficient parameters on our SMS corpus, respectively. So, as
you see the results in the next cell, SVM performing the worst performance
against the other classification algorithms (Naive Bayes and Random Forest)
since we used default parameters.
+ We took a look at their cross validation
scores and performances, as well

```{.python .input  n=4}
classifiers = ['NB', 'SVM', 'RFT']
```

```{.python .input  n=33}
for c in classifiers:
    display('-' * 40 + c + '-' * 40)
    %time sms.validate(c)
```

```{.json .output n=33}
[
 {
  "data": {
   "text/plain": "'----------------------------------------NB----------------------------------------'"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "scores=[ 0.95302013  0.95973154  0.95067265  0.94170404  0.95515695  0.9573991\n  0.94157303  0.94157303  0.94831461  0.93932584]\nmean=0.9488470930132291 std=0.007070744329841395\nCPU times: user 512 ms, sys: 220 ms, total: 732 ms\nWall time: 23.9 s\n"
 },
 {
  "name": "stderr",
  "output_type": "stream",
  "text": "[Parallel(n_jobs=-1)]: Done  10 out of  10 | elapsed:   23.5s finished\n"
 },
 {
  "data": {
   "text/plain": "'----------------------------------------SVM----------------------------------------'"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "scores=[ 0.86353468  0.86353468  0.8632287   0.8632287   0.8632287   0.8632287\n  0.86516854  0.86741573  0.86741573  0.86966292]\nmean=0.8649647070785018 std=0.002242201227467186\nCPU times: user 476 ms, sys: 160 ms, total: 636 ms\nWall time: 25.7 s\n"
 },
 {
  "name": "stderr",
  "output_type": "stream",
  "text": "[Parallel(n_jobs=-1)]: Done  10 out of  10 | elapsed:   25.3s finished\n"
 },
 {
  "data": {
   "text/plain": "'----------------------------------------RFT----------------------------------------'"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "scores=[ 0.97315436  0.95973154  0.95515695  0.9529148   0.96860987  0.97085202\n  0.95730337  0.96404494  0.94831461  0.96629213]\nmean=0.961637459450704 std=0.007820617806919379\nCPU times: user 444 ms, sys: 164 ms, total: 608 ms\nWall time: 25.9 s\n"
 },
 {
  "name": "stderr",
  "output_type": "stream",
  "text": "[Parallel(n_jobs=-1)]: Done  10 out of  10 | elapsed:   25.7s finished\n"
 }
]
```

## Training Stage

+ In training stage, we applied 3 different pipelining with
GridSearchCV on our SMS corpus, respectively.
+ In training stage, to use the
optimum hyperparameters subsequently in testing stage, they were picked by using
GridSearchCV, underneath.
+ After selecting the optimum hyperparameters, each
model corresponding to each pipepline was saved in binary format as a pickle
file. To test any instance on any model, those binary files can be used after
applying deserialization, easily

```{.python .input  n=36}
models = {}

for c in classifiers:
    display('-' * 40 + c + '-' * 40)
    %time models[c] = sms.train(c)
```

```{.json .output n=36}
[
 {
  "data": {
   "text/plain": "'----------------------------------------NB----------------------------------------'"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "data": {
   "text/plain": "'(Grid Search) Best Parameters:'"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "data": {
   "text/html": "<div>\n<style>\n    .dataframe thead tr:only-child th {\n        text-align: right;\n    }\n\n    .dataframe thead th {\n        text-align: left;\n    }\n\n    .dataframe tbody tr th {\n        vertical-align: top;\n    }\n</style>\n<table border=\"1\" class=\"dataframe\">\n  <thead>\n    <tr style=\"text-align: right;\">\n      <th></th>\n      <th>bow__analyzer</th>\n      <th>tfidf__use_idf</th>\n    </tr>\n  </thead>\n  <tbody>\n    <tr>\n      <th>0</th>\n      <td>&lt;bound method SMSBase.create_lemmas of &lt;__main...</td>\n      <td>True</td>\n    </tr>\n  </tbody>\n</table>\n</div>",
   "text/plain": "                                       bow__analyzer  tfidf__use_idf\n0  <bound method SMSBase.create_lemmas of <__main...            True"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "CPU times: user 7.15 s, sys: 364 ms, total: 7.52 s\nWall time: 2min 30s\n"
 },
 {
  "data": {
   "text/plain": "'----------------------------------------SVM----------------------------------------'"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "data": {
   "text/plain": "'(Grid Search) Best Parameters:'"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "data": {
   "text/html": "<div>\n<style>\n    .dataframe thead tr:only-child th {\n        text-align: right;\n    }\n\n    .dataframe thead th {\n        text-align: left;\n    }\n\n    .dataframe tbody tr th {\n        vertical-align: top;\n    }\n</style>\n<table border=\"1\" class=\"dataframe\">\n  <thead>\n    <tr style=\"text-align: right;\">\n      <th></th>\n      <th>classifier__C</th>\n      <th>classifier__kernel</th>\n    </tr>\n  </thead>\n  <tbody>\n    <tr>\n      <th>0</th>\n      <td>100</td>\n      <td>linear</td>\n    </tr>\n  </tbody>\n</table>\n</div>",
   "text/plain": "   classifier__C classifier__kernel\n0            100             linear"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "CPU times: user 6.42 s, sys: 944 ms, total: 7.36 s\nWall time: 2min 32s\n"
 },
 {
  "data": {
   "text/plain": "'----------------------------------------RFT----------------------------------------'"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "data": {
   "text/plain": "'(Grid Search) Best Parameters:'"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "data": {
   "text/html": "<div>\n<style>\n    .dataframe thead tr:only-child th {\n        text-align: right;\n    }\n\n    .dataframe thead th {\n        text-align: left;\n    }\n\n    .dataframe tbody tr th {\n        vertical-align: top;\n    }\n</style>\n<table border=\"1\" class=\"dataframe\">\n  <thead>\n    <tr style=\"text-align: right;\">\n      <th></th>\n      <th>bow__analyzer</th>\n      <th>tfidf__use_idf</th>\n    </tr>\n  </thead>\n  <tbody>\n    <tr>\n      <th>0</th>\n      <td>&lt;bound method SMSBase.create_lemmas of &lt;__main...</td>\n      <td>True</td>\n    </tr>\n  </tbody>\n</table>\n</div>",
   "text/plain": "                                       bow__analyzer  tfidf__use_idf\n0  <bound method SMSBase.create_lemmas of <__main...            True"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "CPU times: user 7.83 s, sys: 1.16 s, total: 8.99 s\nWall time: 3min 18s\n"
 }
]
```

## Testing Stage

+ In testing stage, we tested each pipelining on our test data
that we already splitted into over binary (pickle) file we generated in training
stage.

```{.python .input  n=37}
for c in classifiers:
    display('-' * 40 + c + '-' * 40)
    model_file = '{}_model.pkl'.format(c)
    %time r = sms.test(model_file=model_file)
    display(r)
```

```{.json .output n=37}
[
 {
  "data": {
   "text/plain": "'----------------------------------------NB----------------------------------------'"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "NB_model.pkl file was loaded\nCPU times: user 4.45 s, sys: 12 ms, total: 4.46 s\nWall time: 4.46 s\n"
 },
 {
  "data": {
   "text/plain": "array(['ham', 'ham', 'ham', ..., 'ham', 'ham', 'ham'], \n      dtype='<U4')"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "data": {
   "text/plain": "'----------------------------------------SVM----------------------------------------'"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "SVM_model.pkl file was loaded\nCPU times: user 3.94 s, sys: 0 ns, total: 3.94 s\nWall time: 3.94 s\n"
 },
 {
  "data": {
   "text/plain": "array(['ham', 'ham', 'ham', ..., 'ham', 'ham', 'ham'], \n      dtype='<U4')"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "data": {
   "text/plain": "'----------------------------------------RFT----------------------------------------'"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "RFT_model.pkl file was loaded\nCPU times: user 3.98 s, sys: 8 ms, total: 3.98 s\nWall time: 3.98 s\n"
 },
 {
  "data": {
   "text/plain": "array(['ham', 'ham', 'ham', ..., 'ham', 'ham', 'ham'], \n      dtype='<U4')"
  },
  "metadata": {},
  "output_type": "display_data"
 }
]
```

## Performance Evaluation

+ In this stage, we check out our performance
evaluation by using confusion matrix. We used different metrics, such as
precision, recall, f1 score and accuracy.

+ We also take care about number of
True Positive (TP), True Negative (TN), False Positive (FP), False Negative (FN)
on the confusion matrix.

+ NB is giving the best results according to precision
metric on spam labels on top of test data. Because, your ham sms is not labelled
spam at all. The classification is completely working well if we are focusing
the number of FP (ham vs. spam). Nobody don't want to come across the relevant
(ham) SMS in their spam box. Therefore, the precision of spam is important for
that case because it has the best precision metric (1.00). 

    + Precision =
TP / (TP + FP)
    + Recall = TP / (TP + FN)
    + Accuracy = (TP + TN) / (TP +
TN + FP + FN)
    + F1 = 2 \* Precision \* Recall / (Precision + Recall)
| (Actual)/Pred. |  Ham | Spam |
|---|---|---|
|  Ham | TN  |  FP |
|  Spam | FN
| TP |

+ If we are focusing on the recall metric, some people won't want to see
any spam SMS in their inbox. That case may get also annoying. However, it is not
a big problem like the previous case (number of FP, precision). According to
recall metric on spam labels on top of test data, SVM is giving the best results
since it has the highest recall result (0.81).

```{.python .input  n=40}
for c in classifiers:
    display('-' * 40 + c + '-' * 40)
    model_file = '{}_model.pkl'.format(c)
    model = Util.load_pickle(model_file)
    Util.report_classification(model, 
                               sms._df_train, 
                               sms._df_test, 
                               'context', 
                               'class')
```

```{.json .output n=40}
[
 {
  "data": {
   "text/plain": "'----------------------------------------NB----------------------------------------'"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "--------------------Testing Performance--------------------\n             precision    recall  f1-score   support\n\n        ham       0.95      1.00      0.97       965\n       spam       1.00      0.63      0.77       149\n\navg / total       0.95      0.95      0.95      1114\n\nacc:  0.950628366248\n--------------------Training Performance--------------------\n             precision    recall  f1-score   support\n\n        ham       0.96      1.00      0.98      3860\n       spam       1.00      0.75      0.86       598\n\navg / total       0.97      0.97      0.97      4458\n\nacc:  0.967025572005\n"
 },
 {
  "data": {
   "text/plain": "'----------------------------------------SVM----------------------------------------'"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "--------------------Testing Performance--------------------\n             precision    recall  f1-score   support\n\n        ham       0.97      0.99      0.98       965\n       spam       0.95      0.81      0.87       149\n\navg / total       0.97      0.97      0.97      1114\n\nacc:  0.968581687612\n--------------------Training Performance--------------------\n             precision    recall  f1-score   support\n\n        ham       0.98      0.99      0.98      3860\n       spam       0.92      0.87      0.90       598\n\navg / total       0.97      0.97      0.97      4458\n\nacc:  0.97285778376\n"
 },
 {
  "data": {
   "text/plain": "'----------------------------------------RFT----------------------------------------'"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "--------------------Testing Performance--------------------\n             precision    recall  f1-score   support\n\n        ham       0.96      0.99      0.97       965\n       spam       0.95      0.70      0.81       149\n\navg / total       0.96      0.96      0.95      1114\n\nacc:  0.955116696589\n--------------------Training Performance--------------------\n             precision    recall  f1-score   support\n\n        ham       1.00      1.00      1.00      3860\n       spam       1.00      0.98      0.99       598\n\navg / total       1.00      1.00      1.00      4458\n\nacc:  0.99730820996\n"
 },
 {
  "data": {
   "image/png": "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\n",
   "text/plain": "<matplotlib.figure.Figure at 0x7fb3d6d9b588>"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "data": {
   "image/png": "iVBORw0KGgoAAAANSUhEUgAAAlIAAAEeCAYAAABBrfv5AAAABHNCSVQICAgIfAhkiAAAAAlwSFlz\nAAALEgAACxIB0t1+/AAAIABJREFUeJzt3XlclWX+//HXOYACIgLihqQWKuOCSzGaS+qYpWVlKall\nk2ZaMzo22WKZuWtaapqSWzU2ZaZpVk6azq/cNfcNN3I3MgNFFFFE4Pz+8NsZKRQ4eeTc13k/e5zH\ng3Ofc9/3dWu++VzXfd33bXM4HA5EREREpMjsxd0AEREREatSISUiIiLiIhVSIiIiIi5SISUiIiLi\nIhVSIiIiIi5SISUiIiLiIt/iboCIuK5e1ZZFXmfXsVVuaImISNG4kl/geRmmESkRERERF2lESsTC\nbDZbcTdBRMQlpuSXCikRC7PZNKgsItZkSn6ZcRQiIiIixUAjUiIWZseMoXER8T6m5JcKKRELM2WO\ngYh4H1PyS4WUiIXZDZljICLex5T8UiElYmGm9OhExPuYkl9mlIMiIiIixUAjUiIWZjNksqaIeB9T\n8kuFlIiFmTLHQES8jyn5pUJKxMJMmWMgIt7HlPxSISViYXZDgkhEvI8p+WXGuJqIiIhIMdCIlIiF\n2dQXEhGLMiW/VEiJWJgpcwxExPuYkl8qpEQszJQ5BiLifUzJLzPG1URExBibN28mJiaGM2fOFHdT\nRAqkQspDvP7668TExBATE0PdunWJjo6mbt26zmVTp079w/vYs2cPq1evvu53zp8/z6RJk2jXrh31\n69encePGdO/enRUrVji/s379eqKjo+nfv3++24iPjyc6OpqvvvrqD7dZrs/mwn8iN4q7cuvPf/4z\nCQkJhIaGuty2pKQkBg0aRIsWLahXrx7Nmzenf//+JCYmOr8zZcoUoqOjmT17dr7beOqpp4iOjiYp\nKcnldsi1uZJfnphhKqQ8xKhRo0hISCAhIYFPP/0UgKVLlzqX9enT5w/v47PPPmPNmjXX/PzChQt0\n69aNzZs3M2HCBLZv386yZcto3bo1/fr1c7YLIDQ0lJUrV3Lu3Lk823A4HHz55ZeEh4f/4faKiGe7\nGbnlioMHD9KpUydsNhuffvopO3fuZO7cuQQHB9OlSxd27drl/G65cuVYuHDh77Zx8uRJDhw4cDOb\nLRalQspiDh48yNNPP03jxo2JjY2lf//+pKamOj9/7733aN26NfXr16dVq1bEx8fjcDgYOHAg8+bN\n45NPPiE2Njbfbc+cOZPk5GRmzJhBnTp1sNvthISE0L17d4YOHcrFixed3y1dujQxMTEsXrw4zza2\nbt2Kn58fVapUcc8fgORht9mL/BK5mZKSkoiOjmbu3Lk0bdqUmTNnAlcKroceeoiGDRvSvHlzxo4d\nS05ODgAbN24kOjramW3R0dEsXbqUnj170rBhQ1q3bs2yZcuuuc/hw4dTq1YtRo0aReXKlbHZbERG\nRjJ8+HCeeOIJTp065fzun//8Z44ePcoPP/yQZxtfffUVrVq1usF/GnI1V/LLEzPM81ok13Tx4kV6\n9uxJ3bp1Wb16NcuWLeP8+fMMHjwYuDKvID4+nmnTprFz505mzJjBvHnzWLt2LWPGjKFhw4Z069aN\nLVu25Lv9pUuXEhcXR1BQ0O8+e/TRR+nZs2eeZQ888MDvenJffPEFDzzwwA06YimIzWYr8kukOHz3\n3XcsXryY3r17c+LECV544QX+/ve/s337dv7973/z+eef5zsy9Ktp06bx8ssvs2nTJlq2bMmQIUNw\nOBy/+15qaiqbNm2iR48e+W7npZdeonXr1s73JUuW5J577vndvr/88ktlmZu5kl+emGEqpCxkxYoV\nXLhwgX/+85+ULFmSsmXL0r9/f5YvX05aWhrnzp3DZrM5C6Ho6GhWrVrFXXfdVajt//jjj9x6662F\nbs/999/PgQMHOHToEACZmZn897//5ZFHHin6wYlL7DZbkV8ixeHBBx8kNDQUm81GREQE33//Pffd\ndx8AUVFRxMTEkJCQcM3127VrR61atfDz8+P+++8nLS2N06dP/+57P/74I0CRsqxjx4785z//ITs7\nG4Bdu3Zx6dIlGjduXJRDlCJyJb88McN0+wMLOXr0KOfPn6d+/fp5lttsNn7++WeaN29OkyZNaNu2\nLbGxsTRt2pQOHTpQoUKFQm3fZrM5h9YLIygoiHvvvZfPP/+cAQMG8O2331K7dm0iIiKKdFziOk+c\neCmSn8jIyDzv58+fz/z58zl58iS5ublkZ2fToUOHa65ftWpV58/+/v7Alc7btRQlyxo1akRgYCCr\nVq3i7rvv5osvvuDhhx/2yNEPk5iSXxqRshB/f38iIyOdEzl/fe3du5datWpRsmRJpk2bxsKFC2nS\npAlLly7lvvvuY+/evYXafrVq1Th48GCR2tSpUycWLVpETk4OX3zxBR07dnTl0ETEcH5+fs6fFy5c\nyOTJk3nllVfYsmULCQkJNGvW7Lrr2+2F+3VVrVo1bDZbkbLMZrPxyCOP8MUXX5CVlcWSJUs0si6F\npkLKQqpWrcrJkyfz3FslMzPTOXEyOzubc+fOUbNmTZ599lk+//xzatasWejbENx3330sWLAg3+Hy\nefPm8cwzz/xueaNGjQgICGDZsmUkJCRw7733unh04goTJmqK99mxYwf16tWjdevW+Pn5cfny5d9N\n9nZVmTJlaNasGe+9916+c6heffXVfG/L0LFjR9asWcOyZcuoWbMmt9xyyw1pj1ybJpvLTdeiRQsq\nVKjAqFGjSEtL4/z584wcOZJnn30WgBkzZtC9e3fnHIGkpCRSUlKoVq0acGVEKykpiXPnzuU77P30\n009TpUoVHn/8cTZt2kROTg5paWnMmjWLN954I98e2q89uQkTJnDPPfcQEBDgvj8A+R0TJmqK94mM\njOTo0aOcOnWKlJQUhg0bRlhYGL/88ssN2f6gQYM4fvw4vXr14siRIzgcDpKSkhg8eDCrVq2ibdu2\nv1unYsWKxMbGMmnSJI1G3SSabC43nZ+fH9OmTeP06dO0atWKu+++m3PnzhEfHw9Ar169aNCgAV27\ndqVevXo8+eSTtG/fni5dugAQFxfHxo0bufvuuzl79uzvtu/v78/HH39M27ZtGTx4MLfffjvt27dn\n48aNzJo1yzkx9Lc6duzIiRMnFD7FwISJmuJ9HnvsMWrXrs0999xD165dadasGc8//zy7du3iueee\n+8Pbv+222/j8888pV64cTz75JPXr1+eJJ54gNzeXBQsWEBUVle96cXFxpKam5ltoyY1nymRzmyO/\nsU8RsYQH63cr8jr/2fmJG1oiIlI0ruQXeF6GaURKRERExEW6/YGIhXnifAERkcIwJb9USIlYmCfO\nFxARKQxT8kuFlIiFmXJDOxHxPqbkl9sLqXpVW7p7F1JEWxKu/TwrKT4lgssWeR1PvKeKaZRhnkcZ\n5nm8Ob/MOAoRERGRYqBCSkRERMRFmiMlYmGmXPUiIt7HlPxSISViYaZc9SIi3seU/FIhJWJhplz1\nIiLex5T8UiElYmGm9OhExPuYkl8qpEQkj8zMTOLj48nIyODy5cvExcUREhLC+++/j81mo0qVKvTu\n3RuARYsW8f3332Oz2YiLi+P2228v5taLiDe7dOkS7777LmfPnuXy5ct06tSJqlWrMm3aNLKzs/H1\n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\n",
   "text/plain": "<matplotlib.figure.Figure at 0x7fb38428e0b8>"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "data": {
   "image/png": "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\n",
   "text/plain": "<matplotlib.figure.Figure at 0x7fb38322de48>"
  },
  "metadata": {},
  "output_type": "display_data"
 }
]
```

## Part#2 - Classifying SMS by using Deep Learning with RNN (LSTM)

+ In this
part, we applied deep learning, additionally.
+ We implemented class SMSDL which
is extending from SMSBase, above. This class has some bunch of methods to create
word2vec model over our SMS corpus, and build deep learning layers.
+ During
this experiment,
    + To build a ML model based on Deep Learning, we used Keras
API and its backend is Tensorflow.
    + To apply NLP technique, we used Spacy
libs rather than NLTK, again.
    + To create word2vec model and embedding
vector before applying deep learning, we used Gensim libs instead of using TF-
IDF. We could use Google's, GloVe's, Spacy's pre-trained vectors. However, we
built our word2vec model since we have domain specific data based on SMS.
+ Google's pre-trained vectors based on GoogleNews
        + GLoVe's pre-trained
vectors based on Wikipages
        + Spacy pre-trained vectors
        
+  We
built a deep learning network by using the layers, respectively. We connected
each other layer as you see in the graph, below. 
    + Embedding Layer
    +
Dense Layer
    + LSTM for RNN Layer
    + Dense Layer   
    
<img
src="deeplearning.png">

```{.python .input  n=14}
import gensim, logging

from keras.preprocessing.sequence import pad_sequences
from keras.layers import Dense, LSTM, Embedding, Dropout
from keras.callbacks import EarlyStopping, ModelCheckpoint, TensorBoard
from keras.models import Sequential

from keras.utils.np_utils import to_categorical
from keras.preprocessing.text import Tokenizer
from keras import models

class SMSDL(SMSBase):
    
    # To implement a processing chain, some global variables were used.
    __clean_tokens = None
    __clean_corpus = None
    __word2vec = None
    __tokenizer = None
    __x_train_token_padded_seqs = None  # pad_sequences
    __x_test_token_padded_seqs = None  # pad_sequences
    __embedding_matrix = None
    __nn_layers = None
    
    # To create word2vec, the parameters are used in Gensim, below.
    # There are a few different approaching like the below instead of creating our word2vec model,
    # + Google's pre-trained vectors based on GoogleNews
    # + GLoVe's pre-trained vectors based on Wikipages
    # + Spacy pre-trained vectors
    __embedding_dim = 300
    __window = 15
    __workers = 4
    __cbow_mean = 1
    __alpha = 0.05
    
    # Creating file names for models with respect to given parameters, above.
    __format_word2vec_model = 'emb_dim:{}_window:{}_cbow:{}_apha:{}.bin'
    __word2vec_file = __format_word2vec_model.format(__embedding_dim, __window, __cbow_mean, __alpha)

    # Creating an embedding matrix using by word2vec model and the parameters, below
    __embedding_vector_length = 300
    __max_nb_words = 200000
    __max_input_length = 50

    # Deep Learning Layers' parameters are using to build a deep network. Our network consists of the layers, below:
    # + Embedding Layer
    # + Dense Layer
    # + LSTM for RNN Layer
    # + Dense Layer
    __num_lstm = 100
    __num_dense = 300
    __rate_drop_out = 0.1
    __rate_drop_lstm = float(0.15 + np.random.rand() * 0.25)
    __rate_drop_dense = float(0.15 + np.random.rand() * 0.25)

    # Creating file names for models with respect to given parameters, above.
    __format_dl_model = 'lstm_%d_%d_%.2f_%.2f.h5'
    __model_dl_file = __format_dl_model % (__num_lstm, __num_dense, __rate_drop_lstm, __rate_drop_dense)

    # In training step, those parameters are using, below.
    __number_of_epochs = 100
    __batch_size = 2048
    __validation_split = 0.1
    __shuffle = True
    
    def __init__(self, filename, frac=0.8):
        super().__init__(filename, frac)
        
        self.__x_name = 'context'
        self.__y_name = 'class'
        
        self.__label_classes = {'ham':0, 'spam':1}
        self.__num_classes = len(self.__label_classes)
        self.__encode_labels()
        
        self.__split_sentence_by_lemmas()
        self.__split_sentence_by_tokens()
            

    def __split_sentence_by_lemmas(self):
        self.__sentences_by_lemmas = list(map(lambda c : self.create_lemmas(c), self._df_raw[self.__x_name].values.tolist()))

    def __split_sentence_by_tokens(self):
        self.__sentences_by_tokens = list(map(lambda c : self.create_tokens(c), self._df_raw[self.__x_name].values.tolist()))

    def __encode_labels(self):
        # https://keras.io/utils/#to_categorical
        encoded_list = list(map(lambda c : self.__label_classes[c], self._df_train[self.__y_name].values.tolist()))
        self.__y_train_one_hot = to_categorical(encoded_list, self.__num_classes)
        
    def __create_word2vec(self, by='lemmas'):
        
        if not os.path.exists(self.__word2vec_file) or self.__word2vec is None:
            if by is 'lemmas':
                sentences = self.__sentences_by_lemmas
            elif by is 'tokens':
                sentences = self.__sentences_by_tokens
            else:
                print('You picked wrong function. Please, check your parameter you are using')
                return
            
            logging.basicConfig(format='%(asctime)s : %(levelname)s : %(message)s', level=logging.INFO)
            
            print('{} doesn\'t exist. A new word2vec is being built...'.format(self.__word2vec_file))
            
            self.__word2vec = gensim.models.Word2Vec(sentences,
                                                    size=self.__embedding_dim,
                                                    window=self.__window,
                                                    workers=self.__workers,
                                                    cbow_mean=self.__cbow_mean,
                                                    alpha=self.__alpha)
            self.__word2vec.save(self.__word2vec_file)

        elif self.__word2vec is not None:
            print('{} has already loaded for word2vec...'.format(self.__word2vec_file))
        else:
            print('{} is loading for word2vec...'.format(self.__word2vec_file))
            self.__word2vec = gensim.models.Word2Vec.load(self.__word2vec_file)
        
    def __create_tokenizer(self):
        
        self.__x_train_corpus = self._df_train[self.__x_name].values.tolist() 
        self.__x_test_corpus = self._df_test[self.__x_name].values.tolist()
        
        all_corpus = self.__x_train_corpus + self.__x_test_corpus

        print('x_train_corpus: {}'.format(len(self.__x_train_corpus)))
        print('x_test_corpus: {}'.format(len(self.__x_test_corpus)))

        # https://keras.io/preprocessing/text/#tokenizer
        self.__tokenizer = Tokenizer(num_words=self.__max_nb_words)
        self.__tokenizer.fit_on_texts(all_corpus)

        print('Found %s unique tokens' % len(self.__tokenizer.word_index))


    def __create_sequences(self):

        # https://keras.io/preprocessing/text/#text_to_word_sequence
        x_train_token_seqs = self.__tokenizer.texts_to_sequences(self.__x_train_corpus)
        x_test_token_seqs = self.__tokenizer.texts_to_sequences(self.__x_test_corpus)

        print('x_train_token_seqs: {}'.format(len(x_train_token_seqs)))
        print('x_test_token_seqs: {}'.format(len(x_test_token_seqs)))
        
        # https://keras.io/preprocessing/sequence/#pad_sequences
        self.__x_train_token_padded_seqs = pad_sequences(x_train_token_seqs, maxlen=self.__max_input_length)
        self.__x_test_token_padded_seqs = pad_sequences(x_test_token_seqs, maxlen=self.__max_input_length)
        print('x_train_token_padded_seqs: {}'.format(self.__x_train_token_padded_seqs.shape))
        print('x_test_token_padded_seqs: {}'.format(self.__x_test_token_padded_seqs.shape))

    def __create_embedding_matrix(self):
        
        token_index = self.__tokenizer.word_index
        self.__number_words = min(self.__max_nb_words, len(token_index)) + 1
        
        self.__embedding_matrix = np.zeros((self.__number_words, self.__embedding_vector_length))
        for word, i in token_index.items():
            if word in self.__word2vec.wv.vocab:
                self.__embedding_matrix[i] = self.__word2vec.wv.word_vec(word)

        print('Null word embeddings: %d' % np.sum(np.sum(self.__embedding_matrix, axis=1) == 0))
        print('embedding_matrix: {}'.format(self.__embedding_matrix.shape))
        
    def __init_weights(self, shape, dtype=None):
        print('init_weights shape: {}'.format(shape))
        # assert  shape == embedding_matrix.shape
        return self.__embedding_matrix

    def __create_embedding_layer(self):
        
        # https://keras.io/layers/embeddings/
        embedding = Embedding(self.__number_words,
                                self.__embedding_vector_length,
                                input_length=self.__max_input_length,
                                mask_zero=True,
                                embeddings_initializer=self.__init_weights)
        
        return embedding

    
    def __create_nn_layers(self, weights_filename=None):

        if self.__nn_layers is None:

            self.__nn_layers = Sequential()
            self.__nn_layers.add(self.__create_embedding_layer())

            # https://keras.io/layers/core/#dense
            # https://keras.io/layers/core/#activation
            self.__nn_layers.add(Dense(self.__num_dense, activation='sigmoid'))
            self.__nn_layers.add(Dropout(self.__rate_drop_out))

            # https://keras.io/layers/recurrent/
            self.__nn_layers.add(LSTM(self.__num_lstm, 
                               dropout=self.__rate_drop_lstm, 
                               recurrent_dropout=self.__rate_drop_lstm))
            
            self.__nn_layers.add(Dense(self.__num_classes, activation='softmax'))

            # https://keras.io/metrics/
            self.__nn_layers.compile(loss='binary_crossentropy', optimizer='nadam', metrics=['accuracy'])
            
            self.__nn_layers.summary()
        
        if weights_filename is not None:
            self.__nn_layers.load_weights(weights_filename)

    def __build_model(self, weights_filename=None):

        self.__create_word2vec()
        self.__create_tokenizer()
        self.__create_sequences()
        self.__create_embedding_matrix()
        self.__create_nn_layers(weights_filename)

    def __create_callbacks(self, tensorboard):

        callbacks = []
        # https://keras.io/callbacks/#usage-of-callbacks
        early_stopping = EarlyStopping(monitor='val_acc', patience=10)

        print(self.__model_dl_file)
        # https://keras.io/callbacks/#modelcheckpoint
        model_checkpoint = ModelCheckpoint(self.__model_dl_file, save_best_only=True)

        # https://keras.io/callbacks/#tensorboard
        if tensorboard:
            tensor_board = TensorBoard(log_dir='./logs',
                                       histogram_freq=5,
                                       write_graph=True,
                                       write_images=True,
                                       embeddings_freq=0,
                                       embeddings_layer_names=None,
                                       embeddings_metadata=None)
            callbacks.append(tensor_board)

        callbacks.append(early_stopping)
        callbacks.append(model_checkpoint)

        return callbacks
    
    def load_model(self, filename):
        model = models.load_model(filename)
        return model

    def train(self, tensorboard_enable=False):

        self.__build_model()

        callbacks = self.__create_callbacks(tensorboard_enable)

        # https://keras.io/models/model/
        self._model = self.__nn_layers.fit(self.__x_train_token_padded_seqs,
                             self.__y_train_one_hot,
                             epochs=self.__number_of_epochs,
                             batch_size=self.__batch_size,
                             validation_split=self.__validation_split,
                             shuffle=self.__shuffle,
                             callbacks=callbacks)

        best_val_score = max(self._model.history['val_acc'])
        print('Best Score by val_acc: {}'.format(best_val_score))
        
        self.__df_history = pd.DataFrame()
        self.__df_history['acc'] = self._model.history['acc']
        self.__df_history['loss'] = self._model.history['loss']
        self.__df_history['val_acc'] = self._model.history['val_acc']
        self.__df_history['val_loss'] = self._model.history['val_loss']
        
    
    def __test(self, X_token_padded_seqs):
        prediction_probs = self.__nn_layers.predict(X_token_padded_seqs,
                                             batch_size=self.__batch_size,
                                             verbose=1)

        pre_label_ids = list(map(lambda probs: probs.argmax(), list(prediction_probs)))
        classes = ['ham', 'spam']
    
        return list(map(lambda x: classes[x], pre_label_ids))        
    
            
    def test(self, X_token_padded_seqs=None, weights_filename=None):
        self.__build_model(weights_filename=weights_filename)
        
        if X_token_padded_seqs is None:
            return self.__test(self.__x_test_token_padded_seqs)
        else:
            return self.__test(X_token_padded_seqs)
        
    def __test2(self, X_token_padded_seqs, model):
        prediction_probs = model.predict(X_token_padded_seqs,
                                             batch_size=self.__batch_size,
                                             verbose=1)

        pre_label_ids = list(map(lambda probs: probs.argmax(), list(prediction_probs)))
        classes = ['ham', 'spam']

        return list(map(lambda x: classes[x], pre_label_ids))  
    
    def report_cm2(self, model):        
        X_train = self.__x_train_token_padded_seqs
        X_test = self.__x_test_token_padded_seqs
        
        y_train = self._df_train[self.__y_name].values.tolist()
        y_test = self._df_test[self.__y_name].values.tolist()
        
        y_train_pred = self.__test2(self.__x_train_token_padded_seqs, model)
        y_test_pred = self.__test2(self.__x_test_token_padded_seqs, model)
        
        classes = ['ham', 'spam']
        
        Util.report_cm(y_train, y_test, y_train_pred, y_test_pred, classes)
    
    def report_cm(self, weights_filename):
        self.__build_model(weights_filename)
        
        X_train = self.__x_train_token_padded_seqs
        X_test = self.__x_test_token_padded_seqs
        
        y_train = self._df_train[self.__y_name].values.tolist()
        y_test = self._df_test[self.__y_name].values.tolist()
        
        y_train_pred = self.__test(self.__x_train_token_padded_seqs)
        y_test_pred = self.__test(self.__x_test_token_padded_seqs)
        
        classes = ['ham', 'spam']
        
        Util.report_cm(y_train, y_test, y_train_pred, y_test_pred, classes)

    def display_history(self):
        display(self.__df_history.describe())
        
    def plot_acc(self):
        self.__df_history[['acc', 'val_acc']].plot()
        
    def plot_loss(self):
        self.__df_history[['loss', 'val_loss']].plot()

```

```{.python .input  n=15}
sms_dl = SMSDL('SMSSpam')
```

+ After we splitted messeage into tokens by using Keras' tokenizer, we plotted
CDF graph for frequency of unique words.
+ According to that graph, 
    + the
unique words that appear less than 50 times in 95% of the corpus.
    + the
unique words that appear less than 1 time in the 50% of the corpus.
    + the
unique words that appear less than 4 times in the 75% of the corpus.

```{.python .input  n=23}
tokenizer = sms_dl._SMSDL__tokenizer
freq_of_unique_words = list(map(lambda x : x[1], tokenizer.word_counts.items()))

figure, axes = plt.subplots(1, figsize=(15,5))
Util.plot_cdf(freq_of_unique_words, 
              axes,
              xlim = [0, 50],
              deltay = 0.05,
              ylim = [0, 1.00], xlabel='freq of unique words')
```

```{.json .output n=23}
[
 {
  "data": {
   "text/html": "<div>\n<style>\n    .dataframe thead tr:only-child th {\n        text-align: right;\n    }\n\n    .dataframe thead th {\n        text-align: left;\n    }\n\n    .dataframe tbody tr th {\n        vertical-align: top;\n    }\n</style>\n<table border=\"1\" class=\"dataframe\">\n  <thead>\n    <tr style=\"text-align: right;\">\n      <th></th>\n      <th>freq of unique words</th>\n    </tr>\n  </thead>\n  <tbody>\n    <tr>\n      <th>count</th>\n      <td>9009.000000</td>\n    </tr>\n    <tr>\n      <th>mean</th>\n      <td>9.834832</td>\n    </tr>\n    <tr>\n      <th>std</th>\n      <td>62.019317</td>\n    </tr>\n    <tr>\n      <th>min</th>\n      <td>1.000000</td>\n    </tr>\n    <tr>\n      <th>25%</th>\n      <td>1.000000</td>\n    </tr>\n    <tr>\n      <th>50%</th>\n      <td>1.000000</td>\n    </tr>\n    <tr>\n      <th>75%</th>\n      <td>4.000000</td>\n    </tr>\n    <tr>\n      <th>max</th>\n      <td>2361.000000</td>\n    </tr>\n  </tbody>\n</table>\n</div>",
   "text/plain": "       freq of unique words\ncount           9009.000000\nmean               9.834832\nstd               62.019317\nmin                1.000000\n25%                1.000000\n50%                1.000000\n75%                4.000000\nmax             2361.000000"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "data": {
   "image/png": "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\nAABJRU5ErkJggg==\n",
   "text/plain": "<matplotlib.figure.Figure at 0x7ff7af3eb588>"
  },
  "metadata": {},
  "output_type": "display_data"
 }
]
```

+ After creating a word2vec model, we check top10 similarity of a few arbitrary
words that would be spam keys over our word2vec as you can see in the next
table.

```{.python .input  n=106}
sim = {}
sample_words = ['phone', 'sms', 'bank', 
                'call', 'discount', 'off', 
                'award', 'winner', 'free', 
                'text', 'cash', 'money', 
                'credit', 'prize', 'insurance', 
                'sale', 'click', 'subscriber', 
                '%', '$', 'pound']

for w in sample_words:
    sim[w] = sms_dl._SMSDL__word2vec.most_similar(w)

pd.DataFrame(sim).applymap(lambda x: '{} : {:.2f}'.format(x[0], float(x[1]))).T
```

```{.json .output n=106}
[
 {
  "data": {
   "text/html": "<div>\n<style>\n    .dataframe thead tr:only-child th {\n        text-align: right;\n    }\n\n    .dataframe thead th {\n        text-align: left;\n    }\n\n    .dataframe tbody tr th {\n        vertical-align: top;\n    }\n</style>\n<table border=\"1\" class=\"dataframe\">\n  <thead>\n    <tr style=\"text-align: right;\">\n      <th></th>\n      <th>0</th>\n      <th>1</th>\n      <th>2</th>\n      <th>3</th>\n      <th>4</th>\n      <th>5</th>\n      <th>6</th>\n      <th>7</th>\n      <th>8</th>\n      <th>9</th>\n    </tr>\n  </thead>\n  <tbody>\n    <tr>\n      <th>$</th>\n      <td>january : 0.99</td>\n      <td>g : 0.98</td>\n      <td>th : 0.98</td>\n      <td>% : 0.98</td>\n      <td>bank : 0.98</td>\n      <td>fyi : 0.98</td>\n      <td>dollar : 0.98</td>\n      <td>feb : 0.98</td>\n      <td>ten : 0.97</td>\n      <td>ish : 0.97</td>\n    </tr>\n    <tr>\n      <th>%</th>\n      <td>january : 0.98</td>\n      <td>$ : 0.98</td>\n      <td>gt : 0.98</td>\n      <td>bank : 0.98</td>\n      <td>murder : 0.98</td>\n      <td>th : 0.98</td>\n      <td>your : 0.97</td>\n      <td>face : 0.97</td>\n      <td>g : 0.97</td>\n      <td>@ : 0.97</td>\n    </tr>\n    <tr>\n      <th>award</th>\n      <td>select : 1.00</td>\n      <td>\u00a3 : 0.99</td>\n      <td>draw : 0.99</td>\n      <td>bonus : 0.99</td>\n      <td>show : 0.99</td>\n      <td>reward : 0.99</td>\n      <td>receive : 0.99</td>\n      <td>800 : 0.98</td>\n      <td>prize : 0.98</td>\n      <td>attempt : 0.98</td>\n    </tr>\n    <tr>\n      <th>bank</th>\n      <td>dollar : 0.99</td>\n      <td>$ : 0.98</td>\n      <td>th : 0.98</td>\n      <td>feb : 0.98</td>\n      <td>% : 0.98</td>\n      <td>january : 0.98</td>\n      <td>past : 0.97</td>\n      <td>gt : 0.97</td>\n      <td>fyi : 0.97</td>\n      <td>kill : 0.96</td>\n    </tr>\n    <tr>\n      <th>call</th>\n      <td>0800 : 0.93</td>\n      <td>customer : 0.92</td>\n      <td>please : 0.92</td>\n      <td>mobile : 0.92</td>\n      <td>08712300220 : 0.91</td>\n      <td>arrive : 0.91</td>\n      <td>claim : 0.91</td>\n      <td>camera : 0.90</td>\n      <td>line : 0.90</td>\n      <td>from : 0.90</td>\n    </tr>\n    <tr>\n      <th>cash</th>\n      <td>t&amp;cs : 1.00</td>\n      <td>150ppm : 0.99</td>\n      <td>sae : 0.99</td>\n      <td>subscriber : 0.99</td>\n      <td>000 : 0.98</td>\n      <td>winner : 0.98</td>\n      <td>900 : 0.98</td>\n      <td>1000 : 0.98</td>\n      <td>collection : 0.98</td>\n      <td>operator : 0.98</td>\n    </tr>\n    <tr>\n      <th>click</th>\n      <td>credit : 0.99</td>\n      <td>direct : 0.99</td>\n      <td>connect : 0.99</td>\n      <td>inclusive : 0.99</td>\n      <td>www.comuk.net : 0.99</td>\n      <td>member : 0.99</td>\n      <td>hot : 0.99</td>\n      <td>services : 0.99</td>\n      <td>unsub : 0.99</td>\n      <td>player : 0.99</td>\n    </tr>\n    <tr>\n      <th>credit</th>\n      <td>inclusive : 0.99</td>\n      <td>www.comuk.net : 0.99</td>\n      <td>click : 0.99</td>\n      <td>services : 0.99</td>\n      <td>december : 0.99</td>\n      <td>unsub : 0.99</td>\n      <td>hot : 0.99</td>\n      <td>200 : 0.99</td>\n      <td>&lt; : 0.99</td>\n      <td>account : 0.99</td>\n    </tr>\n    <tr>\n      <th>discount</th>\n      <td>quote : 0.99</td>\n      <td>digital : 0.99</td>\n      <td>150 : 0.99</td>\n      <td>85023 : 0.99</td>\n      <td>ipod : 0.99</td>\n      <td>choose : 0.99</td>\n      <td>chance : 0.99</td>\n      <td>hmv : 0.99</td>\n      <td>app : 0.99</td>\n      <td>official : 0.99</td>\n    </tr>\n    <tr>\n      <th>free</th>\n      <td>handset : 0.97</td>\n      <td>rental : 0.96</td>\n      <td>min : 0.96</td>\n      <td>t&amp;c : 0.96</td>\n      <td>mobile : 0.96</td>\n      <td>network : 0.96</td>\n      <td>uk : 0.95</td>\n      <td>london : 0.95</td>\n      <td>txt : 0.95</td>\n      <td>camera : 0.95</td>\n    </tr>\n    <tr>\n      <th>insurance</th>\n      <td>uncle : 1.00</td>\n      <td>hang : 0.99</td>\n      <td>especially : 0.99</td>\n      <td>jay : 0.99</td>\n      <td>dint : 0.99</td>\n      <td>less : 0.99</td>\n      <td>longer : 0.99</td>\n      <td>weather : 0.99</td>\n      <td>site : 0.99</td>\n      <td>appointment : 0.99</td>\n    </tr>\n    <tr>\n      <th>money</th>\n      <td>why : 0.99</td>\n      <td>bad : 0.99</td>\n      <td>treat : 0.98</td>\n      <td>rent : 0.98</td>\n      <td>may : 0.98</td>\n      <td>something : 0.98</td>\n      <td>way : 0.97</td>\n      <td>even : 0.97</td>\n      <td>bother : 0.97</td>\n      <td>since : 0.97</td>\n    </tr>\n    <tr>\n      <th>off</th>\n      <td>actually : 0.99</td>\n      <td>awesome : 0.99</td>\n      <td>midnight : 0.99</td>\n      <td>probably : 0.99</td>\n      <td>okay : 0.99</td>\n      <td>haven't : 0.99</td>\n      <td>hop : 0.99</td>\n      <td>lemme : 0.99</td>\n      <td>ready : 0.99</td>\n      <td>maybe : 0.99</td>\n    </tr>\n    <tr>\n      <th>phone</th>\n      <td>text : 0.94</td>\n      <td>3510i : 0.92</td>\n      <td>arrive : 0.92</td>\n      <td>video : 0.92</td>\n      <td>week : 0.91</td>\n      <td>0800 : 0.91</td>\n      <td>please : 0.91</td>\n      <td>service : 0.90</td>\n      <td>customer : 0.90</td>\n      <td>to : 0.90</td>\n    </tr>\n    <tr>\n      <th>pound</th>\n      <td>weekly : 0.99</td>\n      <td>1.50 : 0.99</td>\n      <td>mob : 0.99</td>\n      <td>chance : 0.99</td>\n      <td>hmv : 0.99</td>\n      <td>line : 0.99</td>\n      <td>savamob : 0.99</td>\n      <td>standard : 0.99</td>\n      <td>nokia : 0.98</td>\n      <td>pobox : 0.98</td>\n    </tr>\n    <tr>\n      <th>prize</th>\n      <td>900 : 0.99</td>\n      <td>won : 0.99</td>\n      <td>reward : 0.99</td>\n      <td>select : 0.99</td>\n      <td>attempt : 0.98</td>\n      <td>\u00a3 : 0.98</td>\n      <td>award : 0.98</td>\n      <td>draw : 0.98</td>\n      <td>land : 0.98</td>\n      <td>landline : 0.98</td>\n    </tr>\n    <tr>\n      <th>sale</th>\n      <td>log : 0.98</td>\n      <td>list : 0.98</td>\n      <td>access : 0.98</td>\n      <td>budget : 0.98</td>\n      <td>valentine : 0.98</td>\n      <td>space : 0.98</td>\n      <td>tonite : 0.98</td>\n      <td>linerental : 0.98</td>\n      <td>yrs : 0.98</td>\n      <td>available : 0.98</td>\n    </tr>\n    <tr>\n      <th>sms</th>\n      <td>logo : 0.99</td>\n      <td>87077 : 0.98</td>\n      <td>themob : 0.98</td>\n      <td>live : 0.98</td>\n      <td>spook : 0.98</td>\n      <td>wap : 0.98</td>\n      <td>1327 : 0.98</td>\n      <td>s. : 0.98</td>\n      <td>cr9 : 0.98</td>\n      <td>0870 : 0.98</td>\n    </tr>\n    <tr>\n      <th>subscriber</th>\n      <td>cash : 0.99</td>\n      <td>t&amp;cs : 0.99</td>\n      <td>250 : 0.98</td>\n      <td>150ppm : 0.97</td>\n      <td>polyphonic : 0.97</td>\n      <td>operator : 0.97</td>\n      <td>sae : 0.97</td>\n      <td>exciting : 0.97</td>\n      <td>10 : 0.96</td>\n      <td>yr : 0.96</td>\n    </tr>\n    <tr>\n      <th>text</th>\n      <td>phone : 0.94</td>\n      <td>sex : 0.94</td>\n      <td>mobile : 0.94</td>\n      <td>free : 0.93</td>\n      <td>video : 0.93</td>\n      <td>week : 0.92</td>\n      <td>service : 0.92</td>\n      <td>selection : 0.91</td>\n      <td>uk : 0.91</td>\n      <td>60p : 0.90</td>\n    </tr>\n    <tr>\n      <th>winner</th>\n      <td>specially : 0.99</td>\n      <td>000 : 0.99</td>\n      <td>landline : 0.98</td>\n      <td>cash : 0.98</td>\n      <td>won : 0.98</td>\n      <td>900 : 0.98</td>\n      <td>attempt : 0.98</td>\n      <td>t&amp;cs : 0.97</td>\n      <td>1000 : 0.97</td>\n      <td>select : 0.97</td>\n    </tr>\n  </tbody>\n</table>\n</div>",
   "text/plain": "                           0                     1                  2  \\\n$             january : 0.99              g : 0.98          th : 0.98   \n%             january : 0.98              $ : 0.98          gt : 0.98   \naward          select : 1.00              \u00a3 : 0.99        draw : 0.99   \nbank           dollar : 0.99              $ : 0.98          th : 0.98   \ncall             0800 : 0.93       customer : 0.92      please : 0.92   \ncash             t&cs : 1.00         150ppm : 0.99         sae : 0.99   \nclick          credit : 0.99         direct : 0.99     connect : 0.99   \ncredit      inclusive : 0.99  www.comuk.net : 0.99       click : 0.99   \ndiscount        quote : 0.99        digital : 0.99         150 : 0.99   \nfree          handset : 0.97         rental : 0.96         min : 0.96   \ninsurance       uncle : 1.00           hang : 0.99  especially : 0.99   \nmoney             why : 0.99            bad : 0.99       treat : 0.98   \noff          actually : 0.99        awesome : 0.99    midnight : 0.99   \nphone            text : 0.94          3510i : 0.92      arrive : 0.92   \npound          weekly : 0.99           1.50 : 0.99         mob : 0.99   \nprize             900 : 0.99            won : 0.99      reward : 0.99   \nsale              log : 0.98           list : 0.98      access : 0.98   \nsms              logo : 0.99          87077 : 0.98      themob : 0.98   \nsubscriber       cash : 0.99           t&cs : 0.99         250 : 0.98   \ntext            phone : 0.94            sex : 0.94      mobile : 0.94   \nwinner      specially : 0.99            000 : 0.99    landline : 0.98   \n\n                            3                     4                 5  \\\n$                    % : 0.98           bank : 0.98        fyi : 0.98   \n%                 bank : 0.98         murder : 0.98         th : 0.98   \naward            bonus : 0.99           show : 0.99     reward : 0.99   \nbank               feb : 0.98              % : 0.98    january : 0.98   \ncall            mobile : 0.92    08712300220 : 0.91     arrive : 0.91   \ncash        subscriber : 0.99            000 : 0.98     winner : 0.98   \nclick        inclusive : 0.99  www.comuk.net : 0.99     member : 0.99   \ncredit        services : 0.99       december : 0.99      unsub : 0.99   \ndiscount         85023 : 0.99           ipod : 0.99     choose : 0.99   \nfree               t&c : 0.96         mobile : 0.96    network : 0.96   \ninsurance          jay : 0.99           dint : 0.99       less : 0.99   \nmoney             rent : 0.98            may : 0.98  something : 0.98   \noff           probably : 0.99           okay : 0.99    haven't : 0.99   \nphone            video : 0.92           week : 0.91       0800 : 0.91   \npound           chance : 0.99            hmv : 0.99       line : 0.99   \nprize           select : 0.99        attempt : 0.98          \u00a3 : 0.98   \nsale            budget : 0.98      valentine : 0.98      space : 0.98   \nsms               live : 0.98          spook : 0.98        wap : 0.98   \nsubscriber      150ppm : 0.97     polyphonic : 0.97   operator : 0.97   \ntext              free : 0.93          video : 0.93       week : 0.92   \nwinner            cash : 0.98            won : 0.98        900 : 0.98   \n\n                         6                  7                  8  \\\n$            dollar : 0.98         feb : 0.98         ten : 0.97   \n%              your : 0.97        face : 0.97           g : 0.97   \naward       receive : 0.99         800 : 0.98       prize : 0.98   \nbank           past : 0.97          gt : 0.97         fyi : 0.97   \ncall          claim : 0.91      camera : 0.90        line : 0.90   \ncash            900 : 0.98        1000 : 0.98  collection : 0.98   \nclick           hot : 0.99    services : 0.99       unsub : 0.99   \ncredit          hot : 0.99         200 : 0.99           < : 0.99   \ndiscount     chance : 0.99         hmv : 0.99         app : 0.99   \nfree             uk : 0.95      london : 0.95         txt : 0.95   \ninsurance    longer : 0.99     weather : 0.99        site : 0.99   \nmoney           way : 0.97        even : 0.97      bother : 0.97   \noff             hop : 0.99       lemme : 0.99       ready : 0.99   \nphone        please : 0.91     service : 0.90    customer : 0.90   \npound       savamob : 0.99    standard : 0.99       nokia : 0.98   \nprize         award : 0.98        draw : 0.98        land : 0.98   \nsale         tonite : 0.98  linerental : 0.98         yrs : 0.98   \nsms            1327 : 0.98          s. : 0.98         cr9 : 0.98   \nsubscriber      sae : 0.97    exciting : 0.97          10 : 0.96   \ntext        service : 0.92   selection : 0.91          uk : 0.91   \nwinner      attempt : 0.98        t&cs : 0.97        1000 : 0.97   \n\n                             9  \n$                   ish : 0.97  \n%                     @ : 0.97  \naward           attempt : 0.98  \nbank               kill : 0.96  \ncall               from : 0.90  \ncash           operator : 0.98  \nclick            player : 0.99  \ncredit          account : 0.99  \ndiscount       official : 0.99  \nfree             camera : 0.95  \ninsurance   appointment : 0.99  \nmoney             since : 0.97  \noff               maybe : 0.99  \nphone                to : 0.90  \npound             pobox : 0.98  \nprize          landline : 0.98  \nsale          available : 0.98  \nsms                0870 : 0.98  \nsubscriber           yr : 0.96  \ntext                60p : 0.90  \nwinner           select : 0.97  "
  },
  "execution_count": 106,
  "metadata": {},
  "output_type": "execute_result"
 }
]
```

## Training Stage (Deep Learning)
+ We use validation accuracy as an early
stopping in our implementation. When it gets the highest accuracy, it will be
check the next 10 epoch (patience).

```{.python .input  n=16}
sms_dl.train()
```

```{.json .output n=16}
[
 {
  "name": "stderr",
  "output_type": "stream",
  "text": "2017-09-04 17:24:58,869 : INFO : collecting all words and their counts\n2017-09-04 17:24:58,870 : INFO : PROGRESS: at sentence #0, processed 0 words, keeping 0 word types\n2017-09-04 17:24:58,896 : INFO : collected 8544 word types from a corpus of 104594 raw words and 5572 sentences\n2017-09-04 17:24:58,897 : INFO : Loading a fresh vocabulary\n2017-09-04 17:24:58,905 : INFO : min_count=5 retains 1727 unique words (20% of original 8544, drops 6817)\n2017-09-04 17:24:58,906 : INFO : min_count=5 leaves 94271 word corpus (90% of original 104594, drops 10323)\n2017-09-04 17:24:58,912 : INFO : deleting the raw counts dictionary of 8544 items\n2017-09-04 17:24:58,914 : INFO : sample=0.001 downsamples 64 most-common words\n2017-09-04 17:24:58,915 : INFO : downsampling leaves estimated 62143 word corpus (65.9% of prior 94271)\n2017-09-04 17:24:58,916 : INFO : estimated required memory for 1727 words and 300 dimensions: 5008300 bytes\n2017-09-04 17:24:58,926 : INFO : resetting layer weights\n2017-09-04 17:24:58,959 : INFO : training model with 4 workers on 1727 vocabulary and 300 features, using sg=0 hs=0 sample=0.001 negative=5 window=15\n"
 },
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "emb_dim:300_window:15_cbow:1_apha:0.05.bin doesn't exist. A new word2vec is being built...\n"
 },
 {
  "name": "stderr",
  "output_type": "stream",
  "text": "2017-09-04 17:24:59,549 : INFO : worker thread finished; awaiting finish of 3 more threads\n2017-09-04 17:24:59,571 : INFO : worker thread finished; awaiting finish of 2 more threads\n2017-09-04 17:24:59,581 : INFO : worker thread finished; awaiting finish of 1 more threads\n2017-09-04 17:24:59,592 : INFO : worker thread finished; awaiting finish of 0 more threads\n2017-09-04 17:24:59,593 : INFO : training on 522970 raw words (310570 effective words) took 0.6s, 499450 effective words/s\n2017-09-04 17:24:59,597 : INFO : saving Word2Vec object under emb_dim:300_window:15_cbow:1_apha:0.05.bin, separately None\n2017-09-04 17:24:59,599 : INFO : not storing attribute syn0norm\n2017-09-04 17:24:59,602 : INFO : not storing attribute cum_table\n2017-09-04 17:24:59,680 : INFO : saved emb_dim:300_window:15_cbow:1_apha:0.05.bin\n"
 },
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "x_train_corpus: 4458\nx_test_corpus: 1114\nFound 9009 unique tokens\nx_train_token_seqs: 4458\nx_test_token_seqs: 1114\nx_train_token_padded_seqs: (4458, 50)\nx_test_token_padded_seqs: (1114, 50)\nNull word embeddings: 7375\nembedding_matrix: (9010, 300)\ninit_weights shape: (9010, 300)\n_________________________________________________________________\nLayer (type)                 Output Shape              Param #   \n=================================================================\nembedding_2 (Embedding)      (None, 50, 300)           2703000   \n_________________________________________________________________\ndense_3 (Dense)              (None, 50, 300)           90300     \n_________________________________________________________________\ndropout_2 (Dropout)          (None, 50, 300)           0         \n_________________________________________________________________\nlstm_2 (LSTM)                (None, 100)               160400    \n_________________________________________________________________\ndense_4 (Dense)              (None, 2)                 202       \n=================================================================\nTotal params: 2,953,902\nTrainable params: 2,953,902\nNon-trainable params: 0\n_________________________________________________________________\nlstm_100_300_0.33_0.30.h5\nTrain on 4012 samples, validate on 446 samples\nEpoch 1/100\n4012/4012 [==============================] - 17s - loss: 0.6877 - acc: 0.5283 - val_loss: 0.4499 - val_acc: 0.8520\nEpoch 2/100\n4012/4012 [==============================] - 17s - loss: 0.4022 - acc: 0.8659 - val_loss: 0.4136 - val_acc: 0.8520\nEpoch 3/100\n4012/4012 [==============================] - 17s - loss: 0.3798 - acc: 0.8659 - val_loss: 0.3854 - val_acc: 0.8520\nEpoch 4/100\n4012/4012 [==============================] - 18s - loss: 0.3629 - acc: 0.8659 - val_loss: 0.3584 - val_acc: 0.8520\nEpoch 5/100\n4012/4012 [==============================] - 20s - loss: 0.3375 - acc: 0.8662 - val_loss: 0.2977 - val_acc: 0.8520\nEpoch 6/100\n4012/4012 [==============================] - 18s - loss: 0.3053 - acc: 0.8694 - val_loss: 0.2732 - val_acc: 0.8543\nEpoch 7/100\n4012/4012 [==============================] - 18s - loss: 0.2726 - acc: 0.8943 - val_loss: 0.2547 - val_acc: 0.8834\nEpoch 8/100\n4012/4012 [==============================] - 20s - loss: 0.2522 - acc: 0.9093 - val_loss: 0.2476 - val_acc: 0.9126\nEpoch 9/100\n4012/4012 [==============================] - 18s - loss: 0.2086 - acc: 0.9242 - val_loss: 0.1289 - val_acc: 0.9664\nEpoch 10/100\n4012/4012 [==============================] - 19s - loss: 0.1481 - acc: 0.9477 - val_loss: 0.0867 - val_acc: 0.9821\nEpoch 11/100\n4012/4012 [==============================] - 18s - loss: 0.1172 - acc: 0.9629 - val_loss: 0.0896 - val_acc: 0.9753\nEpoch 12/100\n4012/4012 [==============================] - 17s - loss: 0.1049 - acc: 0.9688 - val_loss: 0.0638 - val_acc: 0.9888\nEpoch 13/100\n4012/4012 [==============================] - 18s - loss: 0.0731 - acc: 0.9778 - val_loss: 0.0586 - val_acc: 0.9865\nEpoch 14/100\n4012/4012 [==============================] - 17s - loss: 0.0582 - acc: 0.9835 - val_loss: 0.0696 - val_acc: 0.9798\nEpoch 15/100\n4012/4012 [==============================] - 17s - loss: 0.0524 - acc: 0.9831 - val_loss: 0.0618 - val_acc: 0.9865\nEpoch 16/100\n4012/4012 [==============================] - 18s - loss: 0.0415 - acc: 0.9863 - val_loss: 0.0539 - val_acc: 0.9865\nEpoch 17/100\n4012/4012 [==============================] - 17s - loss: 0.0387 - acc: 0.9860 - val_loss: 0.0490 - val_acc: 0.9910\nEpoch 18/100\n4012/4012 [==============================] - 19s - loss: 0.0327 - acc: 0.9918 - val_loss: 0.0539 - val_acc: 0.9843\nEpoch 19/100\n4012/4012 [==============================] - 19s - loss: 0.0281 - acc: 0.9918 - val_loss: 0.0602 - val_acc: 0.9798\nEpoch 20/100\n4012/4012 [==============================] - 17s - loss: 0.0283 - acc: 0.9910 - val_loss: 0.0538 - val_acc: 0.9843\nEpoch 21/100\n4012/4012 [==============================] - 17s - loss: 0.0234 - acc: 0.9935 - val_loss: 0.0461 - val_acc: 0.9910\nEpoch 22/100\n4012/4012 [==============================] - 16s - loss: 0.0200 - acc: 0.9940 - val_loss: 0.0533 - val_acc: 0.9843\nEpoch 23/100\n4012/4012 [==============================] - 17s - loss: 0.0196 - acc: 0.9935 - val_loss: 0.0466 - val_acc: 0.9888\nEpoch 24/100\n4012/4012 [==============================] - 16s - loss: 0.0165 - acc: 0.9955 - val_loss: 0.0543 - val_acc: 0.9865\nEpoch 25/100\n4012/4012 [==============================] - 17s - loss: 0.0168 - acc: 0.9950 - val_loss: 0.0501 - val_acc: 0.9865\nEpoch 26/100\n4012/4012 [==============================] - 17s - loss: 0.0120 - acc: 0.9963 - val_loss: 0.0527 - val_acc: 0.9865\nEpoch 27/100\n4012/4012 [==============================] - 18s - loss: 0.0107 - acc: 0.9978 - val_loss: 0.0489 - val_acc: 0.9865\nEpoch 28/100\n4012/4012 [==============================] - 20s - loss: 0.0102 - acc: 0.9968 - val_loss: 0.0523 - val_acc: 0.9865\nBest Score by val_acc: 0.9910314083099365\n"
 }
]
```

```{.python .input  n=40}
sms_dl.display_history()
```

```{.json .output n=40}
[
 {
  "data": {
   "text/html": "<div>\n<style>\n    .dataframe thead tr:only-child th {\n        text-align: right;\n    }\n\n    .dataframe thead th {\n        text-align: left;\n    }\n\n    .dataframe tbody tr th {\n        vertical-align: top;\n    }\n</style>\n<table border=\"1\" class=\"dataframe\">\n  <thead>\n    <tr style=\"text-align: right;\">\n      <th></th>\n      <th>acc</th>\n      <th>loss</th>\n      <th>val_acc</th>\n      <th>val_loss</th>\n    </tr>\n  </thead>\n  <tbody>\n    <tr>\n      <th>count</th>\n      <td>28.000000</td>\n      <td>28.000000</td>\n      <td>28.000000</td>\n      <td>28.000000</td>\n    </tr>\n    <tr>\n      <th>mean</th>\n      <td>0.940086</td>\n      <td>0.145030</td>\n      <td>0.949952</td>\n      <td>0.139811</td>\n    </tr>\n    <tr>\n      <th>std</th>\n      <td>0.094867</td>\n      <td>0.169571</td>\n      <td>0.056867</td>\n      <td>0.132794</td>\n    </tr>\n    <tr>\n      <th>min</th>\n      <td>0.528290</td>\n      <td>0.010197</td>\n      <td>0.852018</td>\n      <td>0.046070</td>\n    </tr>\n    <tr>\n      <th>25%</th>\n      <td>0.905533</td>\n      <td>0.022527</td>\n      <td>0.905269</td>\n      <td>0.053173</td>\n    </tr>\n    <tr>\n      <th>50%</th>\n      <td>0.983300</td>\n      <td>0.055283</td>\n      <td>0.984305</td>\n      <td>0.061012</td>\n    </tr>\n    <tr>\n      <th>75%</th>\n      <td>0.993519</td>\n      <td>0.257297</td>\n      <td>0.986547</td>\n      <td>0.249363</td>\n    </tr>\n    <tr>\n      <th>max</th>\n      <td>0.997757</td>\n      <td>0.687664</td>\n      <td>0.991031</td>\n      <td>0.449868</td>\n    </tr>\n  </tbody>\n</table>\n</div>",
   "text/plain": "             acc       loss    val_acc   val_loss\ncount  28.000000  28.000000  28.000000  28.000000\nmean    0.940086   0.145030   0.949952   0.139811\nstd     0.094867   0.169571   0.056867   0.132794\nmin     0.528290   0.010197   0.852018   0.046070\n25%     0.905533   0.022527   0.905269   0.053173\n50%     0.983300   0.055283   0.984305   0.061012\n75%     0.993519   0.257297   0.986547   0.249363\nmax     0.997757   0.687664   0.991031   0.449868"
  },
  "execution_count": 40,
  "metadata": {},
  "output_type": "execute_result"
 }
]
```

## Plotting Accuracy and Learning Curves
+ Plotting accuracy curves and learning
curves for training and validation stage with respect to accuracy and loss.

```{.python .input  n=20}
sms_dl.plot_acc()
sms_dl.plot_loss()
```

```{.json .output n=20}
[
 {
  "data": {
   "image/png": "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\nAAAASUVORK5CYII=\n",
   "text/plain": "<matplotlib.figure.Figure at 0x7ff7b1cd2ef0>"
  },
  "metadata": {},
  "output_type": "display_data"
 },
 {
  "data": {
   "image/png": "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\nAElFTkSuQmCC\n",
   "text/plain": "<matplotlib.figure.Figure at 0x7ff7b1cd2eb8>"
  },
  "metadata": {},
  "output_type": "display_data"
 }
]
```

## Performance Evalution
+ As you recall, we discussed two different important
metrics (precision and recall) in classification apporaches.
+ The both of them
have to be high as possible as to cover two different cases. 
+ According to our
expectations, deep learning approach is giving reasonable and optimal results in
terms of precision an recall metrics on spam labes on top of test data.
+ It has
the highest accuracy than the other classification algortihms (Naive Bayes, SVM,
Random Forest) we already applied since we take care about sequence of word in
each message sematically. TF-IDF is not performing well to cover that
requirement.

```{.python .input  n=21}
sms_dl.report_cm('lstm_100_300_0.33_0.30.h5')
```

```{.json .output n=21}
[
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "emb_dim:300_window:15_cbow:1_apha:0.05.bin has already loaded for word2vec...\nx_train_corpus: 4458\nx_test_corpus: 1114\nFound 9009 unique tokens\nx_train_token_seqs: 4458\nx_test_token_seqs: 1114\nx_train_token_padded_seqs: (4458, 50)\nx_test_token_padded_seqs: (1114, 50)\nNull word embeddings: 7375\nembedding_matrix: (9010, 300)\n4458/4458 [==============================] - 6s     \n1114/1114 [==============================] - 1s\n--------------------Testing Performance--------------------\n             precision    recall  f1-score   support\n\n        ham       0.99      1.00      0.99       971\n       spam       0.97      0.95      0.96       143\n\navg / total       0.99      0.99      0.99      1114\n\nacc:  0.99012567325\n--------------------Training Performance--------------------\n             precision    recall  f1-score   support\n\n        ham       1.00      1.00      1.00      3854\n       spam       0.99      0.98      0.98       604\n\navg / total       1.00      1.00      1.00      4458\n\nacc:  0.995513683266\n"
 },
 {
  "data": {
   "image/png": "iVBORw0KGgoAAAANSUhEUgAAAlIAAAEeCAYAAABBrfv5AAAABHNCSVQICAgIfAhkiAAAAAlwSFlz\nAAALEgAACxIB0t1+/AAAIABJREFUeJzt3XlcVPX+x/HXDIuAiGC4E1qo5IJLkWupuaRlZRlpZWWZ\n3u7VrLRNM3LPSi1LUrO9vGalVpZe7VdqmqWmueBGLqmRmSgQCCIi8/vD29xIFBgZZ8533s8e83jA\nmbN8jz56+zmf+Z4zNofD4UBEREREyszu6QGIiIiIWJUKKREREREXqZASERERcZEKKREREREXqZAS\nERERcZEKKREREREX+Xt6ACLiuqZ1OpR5my37v3HDSEREysaV/ALvyzB1pERERERcpI6UiIXZbDZP\nD0FExCWm5JcKKRELs9nUVBYRazIlv8w4CxEREREPUEdKxMLsmNEaFxHfY0p+qZASsTBT5hiIiO8x\nJb9USIlYmN2QOQYi4ntMyS8VUiIWZsoVnYj4HlPyy4xyUERERMQD1JESsTCbIZM1RcT3mJJfKqRE\nLMyUOQYi4ntMyS8VUiIWZsocAxHxPabklwopEQuzGxJEIuJ7TMkvM/pqIiIiIh6gjpSIhdl0LSQi\nFmVKfqmQErEwU+YYiIjvMSW/VEiJWJgpcwxExPeYkl9m9NVERMQYP/zwA3FxcWRkZHh6KCIlUiHl\nJZ5++mni4uKIi4ujSZMmxMbG0qRJE+ey6dOnn/cxtm3bxsqVK8+5zrFjx5g6dSrdu3enWbNmtGrV\nin79+rF8+XLnOt999x2xsbEMHTq02H0kJSURGxvLZ599dt5jlnOzufCfSHlxV25deeWVJCcnExER\n4fLYUlNTGTlyJO3bt6dp06ZcddVVDB06lJSUFOc606ZNIzY2ltmzZxe7j/vuu4/Y2FhSU1NdHoec\nnSv55Y0ZpkLKS4wfP57k5GSSk5P54IMPAFiyZIlz2aBBg877GB999BGrVq066/u5ubn07duXH374\ngSlTprBx40aWLl1Kp06dGDJkiHNcABEREaxYsYKsrKwi+3A4HHz66adERkae93hFxLtdiNxyxe7d\nu7n11lux2Wx88MEHbN68mblz5xIWFkafPn3YsmWLc92qVauyYMGCM/Zx6NAhdu3adSGHLRalQspi\ndu/ezf3330+rVq2Ij49n6NChpKenO99//fXX6dSpE82aNaNjx44kJSXhcDgYMWIEH374If/+97+J\nj48vdt+zZs3i8OHDvPbaazRu3Bi73U54eDj9+vVj1KhRHD9+3LlupUqViIuLY9GiRUX2sWHDBgIC\nAoiOjnbPH4AUYbfZy/wSuZBSU1OJjY1l7ty5tG3bllmzZgGnC66bbrqJFi1acNVVV/Hcc89x6tQp\nANauXUtsbKwz22JjY1myZAn9+/enRYsWdOrUiaVLl571mGPGjKFhw4aMHz+e2rVrY7PZiIqKYsyY\nMdx1110cOXLEue6VV17Jvn37+Omnn4rs47PPPqNjx47l/Kchf+VKfnljhnnfiOSsjh8/Tv/+/WnS\npAkrV65k6dKlHDt2jMTEROD0vIKkpCRmzJjB5s2bee211/jwww/59ttvmThxIi1atKBv376sX7++\n2P0vWbKEhIQEQkNDz3jvtttuo3///kWW3XDDDWdcyX3yySfccMMN5XTGUhKbzVbml4gnfP311yxa\ntIiBAwdy8OBBhg0bxr/+9S82btzIu+++y/z584vtDP1pxowZPP7446xbt44OHTrwzDPP4HA4zlgv\nPT2ddevWce+99xa7n8cee4xOnTo5f69QoQJdu3Y949iffvqpsszNXMkvb8wwFVIWsnz5cnJzc3n4\n4YepUKECF110EUOHDmXZsmVkZmaSlZWFzWZzFkKxsbF88803XH311aXa/y+//MIll1xS6vFcf/31\n7Nq1iz179gCQl5fHl19+yS233FL2kxOX2G22Mr9EPOHGG28kIiICm81GrVq1+P7777nuuusAiImJ\nIS4ujuTk5LNu3717dxo2bEhAQADXX389mZmZHD169Iz1fvnlF4AyZVmvXr34/PPPKSgoAGDLli2c\nOHGCVq1aleUUpYxcyS9vzDA9/sBC9u3bx7Fjx2jWrFmR5Tabjd9++42rrrqKNm3a0K1bN+Lj42nb\nti09e/akevXqpdq/zWZzttZLIzQ0lGuvvZb58+fzxBNP8NVXX9GoUSNq1apVpvMS13njxEuR4kRF\nRRX5/eOPP+bjjz/m0KFDFBYWUlBQQM+ePc+6fZ06dZw/BwUFAacv3s6mLFnWsmVLQkJC+Oabb+jc\nuTOffPIJN998s1d2P0xiSn6pI2UhQUFBREVFOSdy/vnavn07DRs2pEKFCsyYMYMFCxbQpk0blixZ\nwnXXXcf27dtLtf+6deuye/fuMo3p1ltvZeHChZw6dYpPPvmEXr16uXJqImK4gIAA588LFizglVde\n4cknn2T9+vUkJyfTrl27c25vt5fun6u6detis9nKlGU2m41bbrmFTz75hPz8fBYvXqzOupSaCikL\nqVOnDocOHSrybJW8vDznxMmCggKysrJo0KABDzzwAPPnz6dBgwalfgzBddddx7x584ptl3/44Yf8\n4x//OGN5y5YtCQ4OZunSpSQnJ3Pttde6eHbiChMmaorv2bRpE02bNqVTp04EBARw8uTJMyZ7u6py\n5cq0a9eO119/vdg5VMOHDy/2sQy9evVi1apVLF26lAYNGnDxxReXy3jk7DTZXC649u3bU716dcaP\nH09mZibHjh1j3LhxPPDAAwC89tpr9OvXzzlHIDU1lbS0NOrWrQuc7milpqaSlZVVbNv7/vvvJzo6\nmjvvvJN169Zx6tQpMjMzefvtt3n22WeLvUL780puypQpdO3aleDgYPf9AcgZTJioKb4nKiqKffv2\nceTIEdLS0hg9ejRVqlTh999/L5f9jxw5kgMHDjBgwAB+/vlnHA4HqampJCYm8s0339CtW7cztqlR\nowbx8fFMnTpV3agLRJPN5YILCAhgxowZHD16lI4dO9K5c2eysrJISkoCYMCAATRv3pzbb7+dpk2b\ncs8999CjRw/69OkDQEJCAmvXrqVz58788ccfZ+w/KCiI999/n27dupGYmMjll19Ojx49WLt2LW+/\n/bZzYujf9erVi4MHDyp8PMCEiZrie+644w4aNWpE165duf3222nXrh2PPPIIW7Zs4aGHHjrv/V96\n6aXMnz+fqlWrcs8999CsWTPuuusuCgsLmTdvHjExMcVul5CQQHp6erGFlpQ/Uyab2xzF9T5FxBJu\nbNa3zNt8vvnfbhiJiEjZuJJf4H0Zpo6UiIiIiIv0+AMRC/PG+QIiIqVhSn6pkBKxMG+cLyAiUhqm\n5JcKKRELM+WBdiLie0zJL7cXUk3rdHD3IaSM1ief/fusxHMCwy4q8zbe+EwV0yjDvI8yzPv4cn6Z\ncRYiIiIiHqBCSkRERMRFmiMlYmGm3PUiIr7HlPxSISViYabc9SIivseU/FIhJWJhptz1IiK+x5T8\nUiElYmGmXNGJiO8xJb802VxERETERepIiViYKZM1RcT3uCO/Tpw4wauvvsoff/zByZMnufXWW6lT\npw4zZsygoKAAf39/hgwZQnh4OKtWrWLx4sXYbDa6dOlCp06dKCgoYPr06aSlpWG32xk0aBDVq1c/\n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\n",
   "text/plain": "<matplotlib.figure.Figure at 0x7ff7bb51af60>"
  },
  "metadata": {},
  "output_type": "display_data"
 }
]
```

```{.python .input  n=1}
# sms_dl._SMSDL__nn_layers.save('model.h5')
# model = sms_dl.load_model('model.h5')
# sms_dl.report_cm2(model)
```