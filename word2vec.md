---
title: 
layout: default
---

# Word2Vec

Contents

* <a href="#intro">Introduction</a>
* <a href="#anatomy">Anatomy of Word2Vec</a>
* <a href="#code">Training</a>
* <a href="#windows">Moving Windows</a>
* <a href="#grams">N-grams & Skip-grams</a>
* <a href="#load">Loading Your Data</a>
* <a href="#trouble">Troubleshooting & Tuning Word2Vec</a>
* <a href="#code">A Code Example</a>
* <a href="#next">Next Steps</a>
* <a href="#patent">Word2vec Patent</a>

###<a name="intro">Introduction to Word2Vec</a>

[Deeplearning4j](http://deeplearning4j.org/quickstart.html) implements a distributed form of Word2vec for Java, which works with GPUs. Word2vec was originally conceived by a research team at Google led by Tomas Mikolov. 

Word2vec is a neural net that processes text before that text is handled by deep-learning algorithms. While it does not implement deep learning, Word2vec turns text into a numerical form that deep-learning nets can understand -- the vector. 

(Word2vec's applications extend beyond parsing natural-language sentences occurring in the wild. It can be applied just as well to playlists, social media graphs and other verbal series in which patterns may be discerned.)

Word2vec creates features without human intervention, including the context of individual words. That context comes in the form of multiword windows. Given enough data, usage and context, Word2vec can make highly accurate guesses about a word’s meaning (for the purpose of deep learning, a word's meaning is simply a signal that helps to classify larger entities; e.g. placing a document in a cluster) based on its past appearances. 

Word2vec expects a string of sentences as its input. Each sentence -- that is, each array of words -- is vectorized and  compared to other vectorized lists of words in an n-dimensional vector space. Related words and/or groups of words appear next to each other in that space. Vectorizing them allows us to measure their similarities with some exactitude and cluster them. Those clusters form the basis of search, sentiment analysis and recommendations. 

The output of the Word2vec neural net is a vocabulary with a vector attached to it, which can be fed into a deep-learning net for classification/labeling. 

The skip-gram representation popularized by Mikolov and used in the DL4J implementation has proven to be more accurate than other models due to the more generalizable contexts generated. 

Broadly speaking, we measure words' proximity to each other through their cosine similarity, which gauges the distance/dissimilarity between two word vectors. A perfect 90-degree angle represents identity; i.e. France equals France, while Spain has a cosine distance of 0.678515 from France, the highest of any other country.

Here's a graph of words associated with "China" using Word2vec:

![Alt text](../img/word2vec.png) 

###<a name="anatomy">Anatomy of Word2vec</a>

What do we talk about when we talk about Word2vec? Deeplearning4j's natural-language processing components:

* **SentenceIterator/DocumentIterator**: Used to iterate over a dataset. A SentenceIterator returns strings and a DocumentIterator works with inputstreams. Use the SentenceIterator wherever possible.
* **Tokenizer/TokenizerFactory**: Used in tokenizing the text. In NLP terms, a sentence is represented as a series of tokens. A TokenizerFactory creates an instance of a tokenizer for a "sentence." 
* **VocabCache**: Used for tracking metadata including word counts, document occurrences, the set of tokens (not vocab in this case, but rather tokens that have occurred), vocab (the features included in both bag of words as well as the word vector lookup table)
* **Inverted Index**: Stores metadata about where words occurred. Can be used for understanding the dataset. A Lucene index with the Lucene implementation[1] is automatically created.

Briefly, a two-layer neural net is trained with <a href="../glossary.html#downpoursgd">Gradient Descent</a>. The connection weights for the neural net are of a specified size. <em>syn0</em> in Word2vec terms is the wordvector lookup table, <em>syn1</em> is the activation, and a hierarchical Softmax trains on the two-layer net to calculate the likelihoods of various words being near one another. The Word2vec implementation here uses <a href="../glossary.html#skipgram">skipgrams</a>.

## <a name="code">Training</a> 

Word2Vec trains on raw text. It then records the context, or usage, of each word encoded as word vectors. After training, it's used as lookup table to compose windows of training text for various tasks in natural-language processing.

After lemmatization, Word2vec will conduct automatic multithreaded training based on your sentence data. Then you'll want to save the model. There are a few different components to Word2vec. One of these is the vocab cache. The normal way to save models in deeplearning4j is via the SerializationUtils (java serialization akin to python pickling)

        SerializationUtils.saveObject(vec, new File("mypath"));
       	 
This will save Word2vec to mypath. You can reload it into memory like this:

        Word2Vec vec = SerializationUtils.readObject(new File("mypath"));

You can then use Word2vec as a lookup table in the following way:

        INDArray wordVector = vec.getWordVectorMatrix("myword");
        double[] wordVector = vec.getWordVector("myword");

If the word isn't in the vocabulary, Word2vec returns zeros -- nothing more.

###<a name="windows">Windows</a>

Word2Vec works with neural networks by facilitating the moving-window model for training on word occurrences. There are two ways to get windows for text:

      List<Window> windows = Windows.windows("some text");

This will select moving windows of five tokens from the text (each member of a window is a token).

You also may want to use your own custom tokenizer like this:

      TokenizerFactory tokenizerFactory = new UimaTokenizerFactory();
      List<Window> windows = Windows.windows("text",tokenizerFactory);

This will create a tokenizer for the text, and moving windows based on the tokenizer.

      List<Window> windows = Windows.windows("text",tokenizerFactory);

This will create a tokenizer for the text and create moving windows based on that.

Notably, you can also specify the window size like so:

      TokenizerFactory tokenizerFactory = new UimaTokenizerFactory();
      List<Window> windows = Windows.windows("text",tokenizerFactory,windowSize);

Training word sequence models is done through optimization with the [Viterbi algorithm](https://en.wikipedia.org/wiki/Viterbi_algorithm).

The general idea is to train moving windows with Word2vec and classify individual windows (with a focus word) with certain labels. This could be done for part-of-speech tagging, semantic-role labeling, named-entity recognition and other tasks.

Viterbi calculates the most likely sequence of events (labels) given a transition matrix (the probability of going from one state to another). Here's an example snippet for setup:

<script src="http://gist-it.appspot.com/https://github.com/agibsonccc/java-deeplearning/blob/master/deeplearning4j-examples/src/main/java/org/deeplearning4j/example/word2vec/MovingWindowExample.java?slice=112:121"></script>

From there, each line will be handled something like this:

        <ORGANIZATION> IBM </ORGANIZATION> invented a question-answering robot called <ROBOT>Watson</ROBOT>.

Given a set of text, Windows.windows automatically infers labels from bracketed capitalized text.

If you do this:

        String label = window.getLabel();

on anything containing that window, it will automatically contain that label. This is used in bootstrapping a prior distribution over the set of labels in a training corpus.

The following code saves your Viterbi implementation for later use:

        SerializationUtils.saveObject(viterbi, new File("mypath"));

### <a name="grams">N-grams & Skip-grams</a>

Words are read into the vector one at a time, *and scanned back and forth within a certain range*, much like n-grams. (An n-gram is a contiguous sequence of n items from a given linguistic sequence; it is the nth version of unigram, bigram, trigram, four-gram or five-gram.)  

This n-gram is then fed into a neural network to learn the significance of a given word vector; i.e. significance is defined as its usefulness as an indicator of certain larger meanings, or labels. 

![enter image description here](http://i.imgur.com/SikQtsk.png)

Word2vec uses different kinds of "windows" to take in words: continuous n-grams and skip-grams. 

Consider the following sentence:

    How’s the weather up there?

This can be broken down into a series of continuous trigrams.

    {“How’s”, “the”, “weather”}
    {“the”, “weather”, “up”}
    {“weather”, “up”, “there”}

It can also be converted into a series of skip-grams.

    {“How’s”, “the”, “up”}
    {“the”, “weather”, “there”}
    {“How’s”, “weather”, “up”}
    {“How’s”, “weather”, “there”}
    ...

A skip-gram, as you can see, is a form of discontinous n-gram.

In the literature, you will often see references to a "context window." In the example above, the context window is 3. Many windows use a context window of 5. 

### <a name="dataset">The Dataset</a>

For this example, we'll use a small dataset of articles from the Reuters newswire. 

With DL4J, you can use a **[UimaSentenceIterator](https://uima.apache.org/)** to intelligently load your data. For simplicity's sake, we'll use a **FileSentenceIterator**.

### <a name="load">Loading Your Data</a>

DL4J makes it easy to load a corpus of documents. For this example, we have a folder in the user home directory called "reuters," containing a couple articles.

Consider the following code:

    String reuters= System.getProperty("user.home") +             
    new String("/reuters/");
    File file = new File(reuters);
        
    SentenceIterator iter = new FileSentenceIterator(new SentencePreProcessor() {
    @Override
    public String preProcess(String sentence) {
        return new 
        InputHomogenization(sentence).transform();
        }
    },file);

In lines 1 and 2, we get a file pointer to the directory ‘reuters’. Then we can pass that to FileSentenceIterator. The SentenceIterator is a critical component to DL4J’s Word2Vec usage. This allows us to scan through your data easily, one sentence at a time.

On lines 4-8, we prepare the data by homogenizing it (e.g. lower-case all words and remove punctuation marks), which makes it easier for processing. 

### <a name="prepare">Preparing to Create a Word2Vec Object</a>

Next we need the following

        TokenizerFactory t = new UimaTokenizerFactory();

In general, a tokenizer takes raw streams of undifferentiated text and returns discrete, tidy, tangible representations, which we call tokens and are actually words. Instead of seeing something like: 

    the|brown|fox   jumped|over####spider-man.

A tokenizer would give us a list of words, or tokens, that we can recognize as the following list

1. the
2. brown
3. fox
4. jumped
5. over
6. spider-man

A smart tokenizer will recognize that the hyphen in *spider-man* can be part of the name. 

The word “Uima” refers to an Apache project -- Unstructured Information Management applications -- that helps make sense of unstructured data, as a tokenizer does. It is, in fact, a smart tokenizer. 

### <a name="create">Creating a Word2Vec object</a>

Now we can actually write some code to create a Word2Vec object. Consider the following:

    Word2Vec vec = new Word2Vec.Builder().windowSize(5).layerSize(300).iterate(iter).tokenizerFactory(t).build();

Here we can create a word2Vec with a few parameters

    windowSize : Specifies the size of the n-grams. 5 is a good default

    iterate : The SentenceIterator object that we created earlier
    
    tokenizerFactory : Our UimaTokenizerFactory object that we created earlier

After this line it's also a good idea to set up any other parameters you need.

Finally, we can actually fit our data to a Word2Vec object

    vec.fit();

That’s it. The fit() method can take a few moments to run, but when it finishes, you are free to start querying a Word2Vec object any way you want. 

    String oil = new String("oil");
    System.out.printf("%f\n", vec.similarity(oil, oil));

In this example, you should get a similarity of 1. Word2Vec uses cosine similarity, and a cosine similarity of two identical vectors will always be 1. 

Here are some functions you can call:

1. *similarity(String, String)* - Find the cosine similarity between words
2. *analogyWords(String A, String B, String x)* - A is to B as x is to ?
3. *wordsNearest(String A, int n)* - Find the n-nearest words to A

### <a name="trouble">Troubleshooting & Tuning Word2Vec</a>

*Q: I get a lot of stack traces like this*

       java.lang.StackOverflowError: null
       at java.lang.ref.Reference.<init>(Reference.java:254) ~[na:1.8.0_11]
       at java.lang.ref.WeakReference.<init>(WeakReference.java:69) ~[na:1.8.0_11]
       at java.io.ObjectStreamClass$WeakClassKey.<init>(ObjectStreamClass.java:2306) [na:1.8.0_11]
       at java.io.ObjectStreamClass.lookup(ObjectStreamClass.java:322) ~[na:1.8.0_11]
       at java.io.ObjectOutputStream.writeObject0(ObjectOutputStream.java:1134) ~[na:1.8.0_11]
       at java.io.ObjectOutputStream.defaultWriteFields(ObjectOutputStream.java:1548) ~[na:1.8.0_11]

*A:* Look inside the directory where you started your Word2vec application. This can, for example, be an IntelliJ project home directory or the directory where you typed Java at the command line. It should have some directories that look like:

       ehcache_auto_created2810726831714447871diskstore  
       ehcache_auto_created4727787669919058795diskstore
       ehcache_auto_created3883187579728988119diskstore  
       ehcache_auto_created9101229611634051478diskstore

You can shut down your Word2vec application and try to delete them.

*Q: Not all of the words from my raw text data are appearing in my Word2vec object…*

*A:* Try to raise the layer size via **.layerSize()** on your Word2Vec object like so

        Word2Vec vec = new Word2Vec.Builder().layerSize(300).windowSize(5)
                .layerSize(300).iterate(iter).tokenizerFactory(t).build();

### <a name="code">A Code Example</a>

Now that you have a basic idea of how to set up Word2Vec, here's one example of how it can be used with DL4J's API:

<script src="http://gist-it.appspot.com/https://github.com/deeplearning4j/dl4j-0.0.3.3-examples/blob/master/src/main/java/org/deeplearning4j/word2vec/Word2VecRawTextExample.java?slice=28:97"></script>

There are a couple parameters to pay special attention to here. The first is the number of words to be vectorized in the window, which you enter after WindowSize. The second is the number of nodes contained in the layer, which you'll enter after LayerSize. Those two numbers will be multiplied to obtain the number of inputs. 

Word2Vec is especially useful in preparing text-based data for information retrieval and QA systems, which DL4J implements with [deep autoencoders](../deepautoencoder.html). For sentence parsing and other NLP tasks, we also have an implementation of [recursive neural tensor networks](../recursiveneuraltensornetwork.html).

### <a name="next">Next Steps</a>

An example of sentiment analysis using [Word2Vec is here](http://deeplearning4j.org/sentiment_analysis_word2vec.html). 

(We are still testing our recent implementations of Doc2vec and GLoVE -- watch this space!)

###<a name="patent">Google's Word2vec Patent</a>

Word2vec is [a method of computing vector representations of words](http://arxiv.org/pdf/1301.3781.pdf) introduced by a team of researchers led by Tomas Mikolov in 2013. Google [hosts an open-source version of Word2vec](https://code.google.com/p/word2vec/) released under an Apache 2.0 license. In 2014, Mikolov left Google for Facebook, and in May 2015, [Google was granted a patent for the method](http://patft.uspto.gov/netacgi/nph-Parser?Sect1=PTO2&Sect2=HITOFF&p=1&u=%2Fnetahtml%2FPTO%2Fsearch-bool.html&r=1&f=G&l=50&co1=AND&d=PTXT&s1=9037464&OS=9037464&RS=9037464), which does not abrogate the Apache license under which it has been released.