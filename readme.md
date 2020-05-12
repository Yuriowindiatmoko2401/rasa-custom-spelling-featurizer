<img src="square-logo.svg" width=200 height=200 align="right">

This repository contains a demo project with a custom made tokenizer and featurizer.

It is maintained by Vincent D. Warmerdam, Research Advocate as [Rasa](https://rasa.com/).

# Two Spelling Components
## A Tokenizer and A Featurizer

Rasa offers many useful components to build a digital assistant but 
sometimes you may want to write your own. This document is part
of a series where we will create increasingly complex components from
scratch. In this document we'll build two components that depend on eachother;
a tokenizer that applies a spelling correction and a featurizer that
will generate features telling the model if there's been one. This document
assumes that you're familiar with the contents from the previous guide.

## Example Project

You can clone the repository found [here](https://github.com/RasaHQ/rasa-custom-spelling-featurizer) 
if you'd like to be able to run the same project. The repository contains a relatively small 
rasa project; we're only dealing with four intents and one entity. It's very similar to the project
that was discussed in the previous guide. 

### `data/nlu.md`

```md
## intent:greet
- hey
- hello
...

## intent:goodbye
- bye
- goodbye
...

## intent:bot_challenge
- are you a bot?
- are you a human?
...

## intent:talk_code
- i want to talk about python
- What does this javascript error mean? 
...
```

### `data/stories.md`

```md
## just code
* talk_code
  - utter_proglang

## check bot
* bot_challenge
  - utter_i_am_bot
* goodbye
  - utter_goodbye

## hello and code
* greet
    - utter_greet
* talk_code
    - utter_proglang
```

We'll use the config.yml file that we had before, that means that we have our 
`printer.Printer` at our disposal too. 

## `config.yml`

```yaml
language: en

pipeline:
- name: WhitespaceTokenizer
- name: printer.Printer 
  alias: after tokenizer
- name: CountVectorsFeaturizer
- name: printer.Printer
  alias: after 1st cv
- name: CountVectorsFeaturizer
  analyzer: char_wb
  min_ngram: 1
  max_ngram: 4
- name: printer.Printer
  alias: after 2nd cv
- name: LexicalSyntacticFeaturizer
- name: printer.Printer
  alias: after lexical syntactic featurizer
- name: DIETClassifier
  epochs: 20
- name: printer.Printer
  alias: after diet classifier
policies:
  - name: MemoizationPolicy
  - name: TEDPolicy
  - name: MappingPolicy
```

In this tutorial we're going to make two changes to this pipeline.

1. We're going to replace the `WhitespaceTokenizer` with a custom one called `TextBlobTokenizer`
It will use the `textblob` python package under the hood, hence the name.
It will to handle spelling corrections on our behalf while it is tokenizing. It will also
add special information to the message that can be picked up by a custom featurizer. 
2. We're also going to create a `TextBlobFeaturizer` that will pick up the extra information
generated by the `TextBlobTokenizer` to generate features such that the model is aware when 
a spelling correction is made.

We want to be able to see the effect of these components so we'll re-use the `printer.Printer`
component from the previous guide. This is what the new `config.yml` will look like;

## `config.yml`

```yaml
language: en

pipeline:
- name: printer.Printer 
  alias: after tokenizer
- name: tbfeats.TextBlobTokenizer
- name: CountVectorsFeaturizer
- name: printer.Printer
  alias: after 1st cv
- name: CountVectorsFeaturizer
  analyzer: char_wb
  min_ngram: 1
  max_ngram: 4
- name: printer.Printer
  alias: after 2nd cv
- name: LexicalSyntacticFeaturizer
- name: printer.Printer
  alias: after lexical syntactic featurizer
- name: tbfeats.TextBlobFeaturizer
- name: printer.Printer
  alias: after textblob featurizer
- name: DIETClassifier
  epochs: 20
- name: printer.Printer
  alias: after diet classifier
policies:
  - name: MemoizationPolicy
  - name: TEDPolicy
  - name: MappingPolicy
```

## Creating Tokens 

To create the tokenizer I had a look at the source code of the standard 
[WhiteSpaceTokenizer](https://github.com/RasaHQ/rasa/blob/master/rasa/nlu/tokenizers/whitespace_tokenizer.py) 
inside of Rasa. I kept most items intact but I replaced certain parts with
some code from the [textblob package](). The code for this component looks
like this;

### `tbfeats.py` 

```python
from typing import Text, List
import re
from textblob import TextBlob
from rasa.nlu.training_data import Message
from rasa.nlu.tokenizers.tokenizer import Token, Tokenizer


def split_text(text):
    return re.sub(
            r"[^\w#@&]+(?=\s|$)|"
            r"(\s|^)[^\w#@&]+(?=[^0-9\\s])|"
            r"(?<=[^0-9\s])[^\w._~:/?#\[\]()@!$&*+,;=-]+(?=[^0-9\s])",
            " ",
            text,
        ).split()


class TextBlobTokenizer(Tokenizer):
    language_list = ["en"]
    defaults = {
        # Flag to check whether to split intents
        "intent_tokenization_flag": False,
        # Symbol on which intent should be split
        "intent_split_symbol": "_",
        # Text will be tokenized with case sensitive as default
        "case_sensitive": True,
    }

    def __init__(self, component_config: Dict[Text, Any] = None) -> None:
        """Construct a new tokenizer using the TextBlobTokenizer framework."""
        super().__init__(component_config)
        self.case_sensitive = self.component_config["case_sensitive"]

    @classmethod
    def required_packages(cls) -> List[Text]:
        return ["textblob"]

    def tokenize(self, message: Message, attribute: Text) -> List[Token]:
        orig_text = message.get(attribute)
        if not self.case_sensitive:
            orig_text = orig_text.lower()
        text = str(TextBlob(orig_text).correct())
        orig_words = split_text(orig_text)
        words = split_text(text)

        if not words:
            words = [text]

        message.set("original_words", orig_words)
        return self._convert_words_to_tokens(words, text)
```

There's a few things that we should observe in this code. 

1. The `TextBlobTokenizer` inherits from `rasa.nlu.tokenizers.tokenizer.Tokenizer`.
2. The default `Tokenizer` class that we inherit from has some default settings which we kept intact. 
3. This `TextBlobTokenizer` needs the `textblob` package. Hence we've added it to the `required_packages` method.
4. All the code that handles the spellcheck logic occurs in the `tokenize` method via the `TextBlob` object. If you'd like to check the textblob documentation you can find it [here](https://textblob.readthedocs.io/en/dev/quickstart.html#spelling-correction). 
5. Before the function returns we add custom data to the message via `message.set`. This ensures that we 
are able to retreive the original words that were uttered as well as the ones that received a correction. 
6. We're using the `self._convert_words_to_tokens` method to convert our works to proper tokens. This method
is implemented on the `Tokenize` object that we inherit from and this makes sure that the `Token` objects align
nicely with the text. A `Token` in Rasa has a start and end character that we need to stick to, so it's nice 
that we can use this method to handle all those details correctly. It also handles the creation of `__CLS__` tokens
that represent the entire utterance.
7. We're using the standard `textblob` approach here so we need to make sure that this component is
only used on the English language. Hence we set this explicitly via `language_list = ["en"]`.

If you only added this tokenizer then this is what the `printer.Printer` will show us; 

```
> rasa train
> rasa shell 
> hello werld
text : hello werld
intent: {'name': None, 'confidence': 0.0}
entities: []
original_words: ['hello', 'werld']
tokens: ['hello', 'world', '__CLS__']
```

Notice how `werld` got translated into `world`. 

## Creating Features

It's nice that we've created our own tokenizer but since we're changing the original data
we should be careful. It may be wise to generate some features that will tell the machine
learning model that there's been a spelling correction that's been applied. This way the 
model has an opporunity to correct for it should the spelling correction make a mistake.
This also means that we get to write a `Featurizer`.

In Rasa there's two types of features that you can add to a pipeline; 

- `sparse` features which represent arrays of 0/1 numbers, a `CountVectorizer` generates these
- `dense` features which represent arrays floating numbers, a wordembedding would fall under this category

In this case we're going to be generating dense features. We're going to indicate which words
receive a correction and we're also going to add how confident we are that the current word 
is correct. Not every word will be able to get automatically corrected by `textblob` so having
a metric for confidence should be good to add. 

The relevant code for the `TextBlobFeaturizer` is listed below. 

### `tbfeats.py` 

```python
import typing
from typing import Any, Optional, Text, Dict, List, Type
from textblob import Word
import numpy as np

from rasa.nlu.components import Component
from rasa.nlu.featurizers.featurizer import DenseFeaturizer
from rasa.nlu.config import RasaNLUModelConfig
from rasa.nlu.training_data import Message, TrainingData

if typing.TYPE_CHECKING:
    from rasa.nlu.model import Metadata
from rasa.nlu.constants import DENSE_FEATURE_NAMES, DENSE_FEATURIZABLE_ATTRIBUTES, TEXT


class TextBlobFeaturizer(DenseFeaturizer):
    """A new component"""

    @classmethod
    def required_components(cls) -> List[Type[Component]]:
        """Specify which components need to be present in the pipeline."""
        return [TextBlobTokenizer]

    @classmethod
    def required_packages(cls) -> List[Text]:
        return ["textblob"]

    defaults = {}
    language_list = "en"

    def __init__(self, component_config: Optional[Dict[Text, Any]] = None) -> None:
        super().__init__(component_config)

    def train(
            self,
            training_data: TrainingData,
            config: Optional[RasaNLUModelConfig] = None,
            **kwargs: Any,
    ) -> None:
        for example in training_data.intent_examples:
            for attribute in DENSE_FEATURIZABLE_ATTRIBUTES:
                self._set_textblob_features(example, attribute)

    def _set_textblob_features(self, message: Message, attribute: Text = TEXT):
        tokens = [t.text for t in message.data['tokens'] if t != '__CLS__']
        orig = message.data['original_words']
        correction_made = [t != o for t, o in zip(tokens, orig)]
        correction_made += [any(correction_made)]  # for __CLS__
        confidence = [Word(o).spellcheck()[0][1] for o in orig]
        confidence += [min(confidence)]  # for __CLS__

        X = np.stack([
            np.array(correction_made).astype(np.float),
            np.array(confidence).astype(np.float)
        ]).T
        features = self._combine_with_existing_dense_features(
            message, additional_features=X, feature_name=DENSE_FEATURE_NAMES[attribute]
        )
        message.set(DENSE_FEATURE_NAMES[attribute], features)

    def process(self, message: Message, **kwargs: Any) -> None:
        self._set_textblob_features(message)

    def persist(self, file_name: Text, model_dir: Text) -> Optional[Dict[Text, Any]]:
        # the reason why it is failing is because this is not persisting
        pass

    @classmethod
    def load(
            cls,
            meta: Dict[Text, Any],
            model_dir: Optional[Text] = None,
            model_metadata: Optional["Metadata"] = None,
            cached_component: Optional["Component"] = None,
            **kwargs: Any,
    ) -> "Component":
        """Load this component from file."""

        if cached_component:
            return cached_component
        else:
            return cls(meta)
```

There's a few things to observe about this code. 

1. This `TextBlobFeaturizer` inherits from `DenseFeaturizer`. This will ensure that
all the convenient helper methods are also included.
2. This component depends on the `TextBlobTokenizer` being used in the pipeline. If
this component is missing, we'll get an appropriate error during `rasa train`. Also
notice that we're checking if the `textblob` package is around and if we're still 
using the English language.
3. We're implemented both the `train` as well as the `process` method. 
4. The `train` method needs to typically do two things in featurizers. It needs to 
update the state of the component (this would be relevant for a step where
the component needs to learn from data, 
like [PCA](https://scikit-learn.org/stable/modules/generated/sklearn.decomposition.PCA.html)). 
But it also needs to prepare all the data for the training loop.
5. The `process` method only needs to add the new features per single utterance. Where
the `train` method needs to do this for all the training data this method only needs 
to handle one `message` at a time because it is used during inference. 
6. Because both `train` and `process` do similar things I've added a `_set_textblob_features`
method to handle a single instance which can be used in both methods. 

It's good to zoom in on the `_set_textblob_features` method for a full appreciation of what
happens in there. We've copied the method below and added comments for clarity.

```python
def _set_textblob_features(self, message: Message, attribute: Text = TEXT):
    # we manually grab all the tokens except the `__CLS__` one
    tokens = [t.text for t in message.data['tokens'] if t != '__CLS__']
    # we grab the original words, which did not get a spellcheck 
    orig = message.data['original_words']

    # we make a feature that detects if a correction has been made
    correction_made = [t != o for t, o in zip(tokens, orig)]
    # for the __CLS__ we make a sentence summary if there's any change
    correction_made += [any(correction_made)]  # for __CLS__
    # we make a feature that gives the spelling confidence back
    confidence = [Word(o).spellcheck()[0][1] for o in orig]
    # for the __CLS__ we make a sentence summary via the minimum confidence
    confidence += [min(confidence)]  # for __CLS__

    # we stack the two features together 
    X = np.stack([
        np.array(correction_made).astype(np.float),
        np.array(confidence).astype(np.float)
    ]).T

    # we use `_combine_with_existing_dense_features` to make sure we don't
    # overwrite the data that's attached to the message sofar, note that it
    # is implemented in the `DenseFeaturizer` object that we inherit from 
    features = self._combine_with_existing_dense_features(
        message, additional_features=X, feature_name=DENSE_FEATURE_NAMES[attribute]
    )
    # here we finally add the data to the message. 
    message.set(DENSE_FEATURE_NAMES[attribute], features)
```

You may wonder where the `DENSE_FEATURE_NAMES[attribute]` comes from. It's a bit of
an internal detail this constant allows us to differentiate between features for `intents` 
as well as `responses`. The `DENSE_FEATURE_NAMES` variable is a constant. 

If we now look at what the pipeline produces, we should see some 
dense features appear as well. 

```
> rasa train
> rasa shell 
> hello werld
text : hallo werld
intent: {'name': None, 'confidence': 0.0}
entities: []
original_words: ['hallo', 'werld']
tokens: ['hallo', 'world', '__CLS__']
text_sparse_features: <3x602 sparse matrix of type '<class 'numpy.float64'>'
        with 75 stored elements in COOrdinate format>
text_dense_features: array([[0.        , 1.        ],
                            [1.        , 0.99450549],
                            [1.        , 0.99450549]])
```

These dense features are now passed along to the `DIETClassifier`. 

## Conclusion 

The goal of this document was to demonstrate how components in Rasa work and 
how you might write your own. We should be careful in using this in production
though. It's possible that spelling errors have a bad effect on word embeddings
but it is also possible that the spellchecker makes a translation that is worse. 

Consider this example; 

```
> rasa shell 
> hallo chatbot i wanna buy a pizza

text : hallo chatbot i wanna buy a pizza
intent: {'name': None, 'confidence': 0.0}
entities: []
original_words: ['hallo', 'chatbot', 'i', 'wanna', 'buy', 'a', 'pizza']
tokens: ['hallo', 'charcot', 'i', 'anna', 'buy', 'a', 'penza', '__CLS__']
text_sparse_features: <8x602 sparse matrix of type '<class 'numpy.float64'>'
        with 187 stored elements in COOrdinate format>
text_dense_features: array([[0.        , 1.        ],
                            [1.        , 0.76      ],
                            [0.        , 1.        ],
                            [1.        , 0.99324324],
                            [0.        , 1.        ],
                            [0.        , 1.        ],
                            [1.        , 0.66666667],
                            [1.        , 0.66666667]])
```

In this case the word `wanna` is 'corrected' to `anna` and (worse yet) it even
'corrected' `pizza` to `penza`. The word `hallo` did correctly get translated 
but using a spellcheck in your pipeline is not without risk. 
