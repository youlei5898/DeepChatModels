# Conversation Models in Tensorflow

[NEW MODEL: DynamicBot. More info in next section]

This project is still very much evolving each day, but the core goals are:
* Create a cleaner user interface for tinkering with sequence-to-sequence models and over multiple datasets. Although the release of TensorFlow 1.0 included great improvements in the API for sequence-to-sequence models, there are plenty of further improvements to be made. This project will explore ways to make constructing such models feel more intuitive/customizeable. The ideal result is a chatbot API with the readability of [Keras](https://keras.io/), but with a degree of flexibility closer to TensorFlow.\* For example, the following code is all that is needed (after imports, etc.) to create and train one of the models on the Cornell movie dialogs:
```python
    # (Optional) Number of training samples used per gradient update.
    batch_size = 64
    # (Optional) Specify the max allowed number of words per sentence.
    max_seq_len = 500
    
    # All datasets implement a Dataset interface, found in data/_dataset.py
    dataset = Cornell(FLAGS.vocab_size)

    # Create chat model of choice. Pass in FLAGS values in case you want to change from defaults.
    print("Creating DynamicBot.")
    bot = DynamicBot(dataset,
                     ckpt_dir=FLAGS.ckpt_dir,
                     batch_size=FLAGS.batch_size,
                     state_size=FLAGS.state_size,
                     embed_size=FLAGS.embed_size,
                     learning_rate=FLAGS.learning_rate,
                     lr_decay=FLAGS.lr_decay,
                     is_chatting=FLAGS.decode)


    # Don't forget to compile!
    print("Compiling DynamicBot.")
    bot.compile(max_gradient=FLAGS.max_gradient, reset=FLAGS.reset_model)

    # Train an epoch on the data. CTRL-C at any time to safely stop training.
    # Model saved in FLAGS.ckpt_dir if specified, else "./out"
    if not FLAGS.decode:
        print("Training bot. CTRL-C to stop training.")
        bot.train(dataset.train_data, dataset.valid_data,
                  nb_epoch=FLAGS.nb_epoch,
                  steps_per_ckpt=FLAGS.steps_per_ckpt)

    else:
        print("Initiating chat session")
        bot.decode()
```

* Explore how [personalities of chatbots](https://arxiv.org/pdf/1603.06155.pdf) change when trained on different datasets, and methods for improving speaker consistency.
* Add support for "teacher mode": an interactive chat session where the user can tell the bot how well they're doing, and suggest better responses that the bot can learn from.


\*Any similarities in naming conventions to Keras are just that. No Keras source code was viewed/consulted for this project.

## Faster Embedding, Encoding, and Chatting

The newest model, ```DynamicBot```, is substantially faster than the previous models (bucketed models in legacy_models.py). Here are some of the key design differences for comparison:

|               | BucketModel | DynamicBot |
| ------------  | ----------    | -----------------------   |
| Embedding     | Used TensorFlow's ```EmbeddingWrapper```, which computes the embedding on a batch at each timestep. | Uses custom Embedder class to dynamically embed full batch-concatenated inputs of variable sequence length. |
| Encoding      | Employed the standard 'bucketed' model as described in TensorFlow sequence-to-sequence tutorial. Requires inputs to be padded to the same sequence length, for each bucket, which can result in unnecessarily large matrices of mainly zeros. | Combines the functionality of the new dynamic_rnn method in Tensorflow r1.0, wrapped inside a custom Encoder class. Input sequences are first fed to a custom batch_padded preprocessing utility (see utils/io_utils) that drastically reduces the occurrence of zero-padded sequences and allows for variable-length sequence batches. |
| Chatting      | Requires output to be assigned to a bucket, which constrains the raw output sequences to be constrained to pre-defined lengths. They then have to be truncated to remove padding. | Responses are generated naturally: once DynamicBot has read your input, it writes its response word by word until it signals that it's done speaking. No awkward post-processing required, and faster response times. |

One particular feature of DynamicBot worth mentioning is that the output generation and sampling process is _fully contained within the graph_ structure itself. This is in contrast with standard methods of outputting large arrays representing the logits (unnormalized log probabilities) and then sampling/argmax-ing over these. DynamicBot, however, directly returns its generated responses as a sequence of word-tokens.

At present, I'm running longer training/chatting sessions on all models to eventually report quantitative comparisons. They will be reported here after all models have given it their best shot.



## Project Overview

(TODO: needs updating)
The following have been more-or-less completed. 

* Models:
    * Rewrite conversation model with faster embedding technique and new TF support for dynamic unrolling.
    * Implement an attention-based embedding sequence-to-sequence model with the help of the tensorflow.contrib libraries.
    * Implement a simpler embedding sequence-to-sequence from "scratch" (minimal use of contrib).
* Datasets:
    * **WMT'15** : English-to-French translation.
    * **Ubuntu Dialogue Corpus**: Reformatted as single-turn to single-response pairs.
    * **Cornell Movie-Dialogs**: Recently (March 5) incorporated [this preprocessed](https://github.com/suriyadeepan/datasets/tree/master/seq2seq/cornell_movie_corpus) version of the Cornell corpus. I'll be processing and reformatting it further.
