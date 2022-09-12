# Using cosine similarity for commands filtering

## Cosine similarity
In this chapter I will show how cosine similarity can be used to retrieve similar (or dissimilar) sequences of characters. This technique is widely used in natural language processing (NLP). Cosine similarity can be used for comparing documents, it gives a measure for the angle between vectors of data. <br>
It is based on the dot product, which is a good operator for yielding high values if two vectors have high values in the same dimensions. Because it favors long vectors, it may be a big disadvantage for the analysis and therefore the outcome must be normalized. The dot product is divided by the lengths of the two vectors being compared\[1\]. The following formula is used for computing the cosine similarity for two vectors (v and w):

$$cosine(v, w) = \frac{v \cdot w}{\lVert v \rVert \lVert w \rVert} = \frac{\sum^{N}_{i=1}v_{i}w_{i}}{\sum^{N}_{i=1}v_{i}^2 \sum^{N}_{i=1}w_{i}^2}$$

Cosine similarity can be thought as an angle between two vectors being compared. In fact, after computing cosine distance, we can use arc-cosine function to translate it directly into an angle, so that the values are in the range of 0-180 degrees. Two vectors with the same direction have cosine similarity equal to 0. Symmetry is quite an obvious characteristic of the cosine similarity, since there will be the same distance from v to w and from w to v [2].

##### Global imports


```python
import numpy as np
import os
import pandas as pd

from sklearn.metrics.pairwise import cosine_similarity
```

##### Local imports


```python
from src.data_cleaning.datasetBuilder import DatasetBuilder
from src.data_cleaning.datasetCleaner import DatasetCleaner
from src.time_analysis.timeStepPreprocessor import TimeStepPreprocessor
```

First, the data set is cleaned an the time attribute is split into the time and date.


```python
data = DatasetBuilder("src/data").buildDataset()
tsp = TimeStepPreprocessor(data, "TimeCreated")
tsp.preprocessTimeSteps()
data = tsp.preprocessed_date_time
cleaner = DatasetCleaner(data, "Message")
cleaned_data = cleaner.removeNullValues(generate_report=False)
```

The most interesting attribute for this analysis is the *Message* attribute, since it contains the log messages we try to filter. In NLP the entire set of documents is called a corpus, thus this will be the variable to hold the messages' contents.


```python
corpus = cleaned_data["Message"].to_numpy()
```

Now, there are a few steps to complete before proceeding with cosine similarity computation. <br>
1) The messages lengths must be obtained. <br>
2) Based on their lengths, they must be grouped according to length intervals. <br>
3) The messages' characters must be represented in a numeric format and the differences in lengths among messages belonging to the same group must be eliminated by adding additional zeros to the vectors. Therefore, the lengths of messages in each group will correspond to the length of the longest message in that group.

## Some basic functions to be used

The first important function takes an array of texts, converts it into a Pandas Series and assigns lengths of the messages to the new attribute. The output of this function is a dataframe containing the message's length and its contents stored in another attribute.


```python
def getLengths(list_texts):
    series_text = pd.Series(list_texts)
    lengths = series_text.apply(len)
    dataframe = pd.DataFrame(lengths)
    dataframe.columns = ["MessageLength"]
    dataframe = dataframe.reset_index()
    dataframe = dataframe.drop(columns=["index"])
    dataframe.columns = ["MessageLength"]
    dataframe = dataframe.assign(Text=series_text.to_numpy())
    return dataframe
```

Another function searches for the message of the greatest length.


```python
def getMaxLength(series_text):
    max_len = 0
    for l in series_text.apply(len):
        if l > max_len:
            max_len = l
    return max_len
```

The function convertToNumeric takes a message and converts characters to numbers. It is done using ord() function, which returns Unicode for a character. <br>
Another argument it takes is the maximum length, so it can compute the difference between the message's length and the max length and appends zeros to the vector of numbers representing the message. <br>
The function returns a list of numbers which represent the message as a numeric vector.

The reason why the vectors are obtained this way - using the characters' Unicode representation, is the fact that system commands and system logs are a very specific kind of language. Some signatures or commands flags can differ by a single letter, so if we have two vectors of system commands differing only by some flags given, such a difference will be few numbers. This is an alternative technique for word-oriented techniques - such as BoW or TF-IDF. Therefore, we can measure the difference at individual characters level for sequences of characters, rather than measuring the occurrences of words in each text for instance.


```python
def convertToNumeric(text, max_length):
    numeric_representation = [ord(ch) for ch in text]
    diff = abs(len(numeric_representation) - max_length)
    for i in range(diff):
        numeric_representation.append(0)
    return numeric_representation
```

The function constructNumericRepresentation is a helper function which applies the getMaxLength and convertToNumeric functions to a Series of texts.


```python
def constructNumericRepresentation(messages_series):
    numerical_arrays = []
    max_len = getMaxLength(messages_series)
    for text in messages_series:
        numerical_arrays.append(convertToNumeric(text, max_len))
    return numerical_arrays
```

## Group by lengths

So the first task is to group messages according to length intervals.

The function getLowerBounds is a helper function which establishes the lower limits for the intervals in the context of the entire dataset. It runs from the message of the smallest length, makes a step equal to the width of an interval - provided as the function's argument and stops as soon as it reaches the value of the maximum length + the interval's width.


```python
def getLowerBounds(text_series, interval_width=10):
    # prepare a list of lower bound values for grouping messages according to their length category.
    # the groups will be 10 characters wide.
    df_temp = getLengths(text_series.unique())
    start = df_temp["MessageLength"].min()
    step = interval_width
    stop = df_temp["MessageLength"].max() + interval_width
    lower_bounds = []
    for i in range(start, stop, step):
        lower_bounds.append(i)
    return lower_bounds
```

The function assignToGroups takes the messages as the input and the interval's width as a named argument (keyword argument) which by default is set to 10. It can be optimized according to the needs. The function executes in quadratic time complexity ($O^{2}$), however, it only appends to a list of groups based on a condition given.


```python
def assignToGroups(text_series, interval_width=10):
    # assign the messages to groups
    df_temp = getLengths(text_series.unique())
    lower_bounds = getLowerBounds(text_series, interval_width)
    groups = []
    for i in range(len(lower_bounds)-1):
        group = []
        for j in range(len(df_temp)):
            # if a message is in range in terms of its length, add it.
            if (df_temp.MessageLength.iloc[j] >= lower_bounds[i]) & (df_temp.MessageLength.iloc[j] < lower_bounds[i+1]):
                group.append(df_temp.Text.iloc[j])
        # append the group of messages to the global group of groups
        groups.append(group)
    return groups
```

It is import to extract groups which are non-empty, as some groups may contain no records. This is because the messages did not meet the criteria of belonging to a group - they were too short or too long.


```python
def extractNonEmptyGroups(groups):
    non_empty_groups = []
    for g in groups:
        if len(g) > 0:
            g = pd.Series(g)
            non_empty_groups.append(g)
    return non_empty_groups
```


```python
non_empty_groups = extractNonEmptyGroups(assignToGroups(cleaned_data["Message"], interval_width=10))
len(non_empty_groups)
```




    125



## Get numeric representation for each group

Once we have groups which are non-empty lists, we can obtain the numeric representation of the messages for each group. Splitting the process into grouping texts and retrieving their numeric representations allows for mapping the numeric groups to the text groups.


```python
def createNumericGroups(groups_not_empty):
    numeric_groups = []
    for g in groups_not_empty:
        numeric_groups.append(constructNumericRepresentation(g))
    return numeric_groups
```

Since the function createNumericGroups executes in quadratic time complexity ($O^{2}$) due to a nested loop resulting in applying the constructNumericRepresentation function in a for loop, this is another bottleneck for the program.


```python
numeric_groups = createNumericGroups(non_empty_groups)
```


```python
len(numeric_groups)
```




    125



## Get cosine similarity matrix for each group

The final step is to compute the cosine similarity for vectors of numeric representation of characters and select the most representative and the least representative ones within each group.

The following function computes a pairwise cosine similarity for groups of messages. The scores are returned in a form of a matrix and for each message an average similarity is calculated so that the message with the highest and the lowest score can be selected for each group. A list of tuples is retrurned and each tuple contains information about the message's index, the most representative message and the least representative message.


```python
def findRepresentativeMessages(numeric_groups, literal_groups):
    # the arrays must be the same size, otherwise retrieval of the literal message would not be possible
    assert len(numeric_groups) == len(literal_groups), "Arrays of unequal length!"
    representative_messages = []
    for i in range(len(numeric_groups)):
        # in case that there is only a single message in the group, append it as it is the
        # only representation of the group
        if len(literal_groups[i]) == 1:
            representative_messages.append((i, literal_groups[i][0], None))
        # otherwise, determine the cosine similarity scores for the messages
        # and get the most representative and the least representative ones
        else:
            # obtain cosine similarity matrix
            cosine_df = pd.DataFrame(cosine_similarity(numeric_groups[i], dense_output=True))
            # get the scores of similarity to other commands
            scores = pd.Series((cosine_df.sum(axis=1) - 1) / (len(cosine_df) - 1))

            # There should be no error, as the groups of just 1 element have been filtered
            # however, if there is still an error, continue.
            if (np.isnan(scores.idxmax())) | (np.isnan(scores.idxmin())):
                continue
            else:
                # return the most representative and least representative commands for that group
                representative_messages.append((i, literal_groups[i][scores.idxmax()], literal_groups[i][scores.idxmin()]))

    return representative_messages
```


```python
filtered_messages = findRepresentativeMessages(numeric_groups, non_empty_groups)
```

A helper function can be created to save the results in a text file. 


```python
def saveExtractedMessages(extracted_messages, output_directory):
    status = 1
    try:
        if not os.path.isdir(output_directory):
            os.mkdir(output_directory)
        with open(f"{output_directory}/most_representative_messages.txt", 'w') as fh:
            for record in extracted_messages:
                fh.write(f"\n\n\nRECORD: {record[0]}\n\n{record[1]}\n\n\n")
        with open(f"{output_directory}/least_representative_messages.txt", 'w') as fh:
            for record in extracted_messages:
                fh.write(f"\n\n\nRECORD: {record[0]}\n\n{record[2]}\n\n\n")
        status = 0

    except Exception as ex:
        print("Error: %s.\nSaving messages failed." % ex)

    return status
```


```python
saveExtractedMessages(filtered_messages, "filtered_messages")
```




    0



Having the outputs saved to a text file, we can filter through interesting messages on the initial dataframe and explore event information.


```python
data.iloc[[44238, 44267, 44273]]
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Message</th>
      <th>Id</th>
      <th>Version</th>
      <th>Qualifiers</th>
      <th>Level</th>
      <th>Task</th>
      <th>Opcode</th>
      <th>Keywords</th>
      <th>RecordId</th>
      <th>ProviderName</th>
      <th>...</th>
      <th>ContainerLog</th>
      <th>MatchedQueryIds</th>
      <th>Bookmark</th>
      <th>LevelDisplayName</th>
      <th>OpcodeDisplayName</th>
      <th>TaskDisplayName</th>
      <th>KeywordsDisplayNames</th>
      <th>Properties</th>
      <th>Date</th>
      <th>Time</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>44238</th>
      <td>A Terminal Services session change was process...</td>
      <td>1006</td>
      <td>0.0</td>
      <td>NaN</td>
      <td>4</td>
      <td>0</td>
      <td>0.0</td>
      <td>-9223372036854775776</td>
      <td>61</td>
      <td>Microsoft-Windows-Wcmsvc</td>
      <td>...</td>
      <td>c:\users\hyphens\hackthebox\event_horizon\proj...</td>
      <td>System.UInt32[]</td>
      <td>System.Diagnostics.Eventing.Reader.EventBookmark</td>
      <td>Information</td>
      <td>Info</td>
      <td>NaN</td>
      <td>System.Collections.ObjectModel.ReadOnlyCollect...</td>
      <td>System.Collections.Generic.List`1[System.Diagn...</td>
      <td>23/10/2020</td>
      <td>11:55:26</td>
    </tr>
    <tr>
      <th>44267</th>
      <td>A Terminal Services session change was process...</td>
      <td>1006</td>
      <td>0.0</td>
      <td>NaN</td>
      <td>4</td>
      <td>0</td>
      <td>0.0</td>
      <td>-9223372036854775776</td>
      <td>32</td>
      <td>Microsoft-Windows-Wcmsvc</td>
      <td>...</td>
      <td>c:\users\hyphens\hackthebox\event_horizon\proj...</td>
      <td>System.UInt32[]</td>
      <td>System.Diagnostics.Eventing.Reader.EventBookmark</td>
      <td>Information</td>
      <td>Info</td>
      <td>NaN</td>
      <td>System.Collections.ObjectModel.ReadOnlyCollect...</td>
      <td>System.Collections.Generic.List`1[System.Diagn...</td>
      <td>22/10/2020</td>
      <td>15:38:59</td>
    </tr>
    <tr>
      <th>44273</th>
      <td>A Terminal Services session change was process...</td>
      <td>1006</td>
      <td>0.0</td>
      <td>NaN</td>
      <td>4</td>
      <td>0</td>
      <td>0.0</td>
      <td>-9223372036854775776</td>
      <td>26</td>
      <td>Microsoft-Windows-Wcmsvc</td>
      <td>...</td>
      <td>c:\users\hyphens\hackthebox\event_horizon\proj...</td>
      <td>System.UInt32[]</td>
      <td>System.Diagnostics.Eventing.Reader.EventBookmark</td>
      <td>Information</td>
      <td>Info</td>
      <td>NaN</td>
      <td>System.Collections.ObjectModel.ReadOnlyCollect...</td>
      <td>System.Collections.Generic.List`1[System.Diagn...</td>
      <td>22/10/2020</td>
      <td>15:30:56</td>
    </tr>
  </tbody>
</table>
<p>3 rows × 28 columns</p>
</div>




```python
data.iloc[[44238, 44267, 44273]]
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Message</th>
      <th>Id</th>
      <th>Version</th>
      <th>Qualifiers</th>
      <th>Level</th>
      <th>Task</th>
      <th>Opcode</th>
      <th>Keywords</th>
      <th>RecordId</th>
      <th>ProviderName</th>
      <th>...</th>
      <th>ContainerLog</th>
      <th>MatchedQueryIds</th>
      <th>Bookmark</th>
      <th>LevelDisplayName</th>
      <th>OpcodeDisplayName</th>
      <th>TaskDisplayName</th>
      <th>KeywordsDisplayNames</th>
      <th>Properties</th>
      <th>Date</th>
      <th>Time</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>44238</th>
      <td>A Terminal Services session change was process...</td>
      <td>1006</td>
      <td>0.0</td>
      <td>NaN</td>
      <td>4</td>
      <td>0</td>
      <td>0.0</td>
      <td>-9223372036854775776</td>
      <td>61</td>
      <td>Microsoft-Windows-Wcmsvc</td>
      <td>...</td>
      <td>c:\users\hyphens\hackthebox\event_horizon\proj...</td>
      <td>System.UInt32[]</td>
      <td>System.Diagnostics.Eventing.Reader.EventBookmark</td>
      <td>Information</td>
      <td>Info</td>
      <td>NaN</td>
      <td>System.Collections.ObjectModel.ReadOnlyCollect...</td>
      <td>System.Collections.Generic.List`1[System.Diagn...</td>
      <td>23/10/2020</td>
      <td>11:55:26</td>
    </tr>
    <tr>
      <th>44267</th>
      <td>A Terminal Services session change was process...</td>
      <td>1006</td>
      <td>0.0</td>
      <td>NaN</td>
      <td>4</td>
      <td>0</td>
      <td>0.0</td>
      <td>-9223372036854775776</td>
      <td>32</td>
      <td>Microsoft-Windows-Wcmsvc</td>
      <td>...</td>
      <td>c:\users\hyphens\hackthebox\event_horizon\proj...</td>
      <td>System.UInt32[]</td>
      <td>System.Diagnostics.Eventing.Reader.EventBookmark</td>
      <td>Information</td>
      <td>Info</td>
      <td>NaN</td>
      <td>System.Collections.ObjectModel.ReadOnlyCollect...</td>
      <td>System.Collections.Generic.List`1[System.Diagn...</td>
      <td>22/10/2020</td>
      <td>15:38:59</td>
    </tr>
    <tr>
      <th>44273</th>
      <td>A Terminal Services session change was process...</td>
      <td>1006</td>
      <td>0.0</td>
      <td>NaN</td>
      <td>4</td>
      <td>0</td>
      <td>0.0</td>
      <td>-9223372036854775776</td>
      <td>26</td>
      <td>Microsoft-Windows-Wcmsvc</td>
      <td>...</td>
      <td>c:\users\hyphens\hackthebox\event_horizon\proj...</td>
      <td>System.UInt32[]</td>
      <td>System.Diagnostics.Eventing.Reader.EventBookmark</td>
      <td>Information</td>
      <td>Info</td>
      <td>NaN</td>
      <td>System.Collections.ObjectModel.ReadOnlyCollect...</td>
      <td>System.Collections.Generic.List`1[System.Diagn...</td>
      <td>22/10/2020</td>
      <td>15:30:56</td>
    </tr>
  </tbody>
</table>
<p>3 rows × 28 columns</p>
</div>




```python
data.iloc[33632]
```




    Message                 An account was logged off.\r\n\r\nSubject:\r\n...
    Id                                                                   4634
    Version                                                               0.0
    Qualifiers                                                            NaN
    Level                                                                   0
    Task                                                                12545
    Opcode                                                                0.0
    Keywords                                             -9214364837600034816
    RecordId                                                              509
    ProviderName                          Microsoft-Windows-Security-Auditing
    ProviderId                           54849625-5478-4994-a5ba-3e3b0328c30d
    LogName                                                          Security
    ProcessId                                                           824.0
    ThreadId                                                           4960.0
    MachineName                                               WIN-PTFPHFCRDJ0
    UserId                                                                NaN
    ActivityId                                                            NaN
    RelatedActivityId                                                     NaN
    ContainerLog            c:\users\hyphens\hackthebox\event_horizon\proj...
    MatchedQueryIds                                           System.UInt32[]
    Bookmark                 System.Diagnostics.Eventing.Reader.EventBookmark
    LevelDisplayName                                              Information
    OpcodeDisplayName                                                    Info
    TaskDisplayName                                                    Logoff
    KeywordsDisplayNames    System.Collections.ObjectModel.ReadOnlyCollect...
    Properties              System.Collections.Generic.List`1[System.Diagn...
    Date                                                           22/10/2020
    Time                                                             15:30:58
    Name: 33632, dtype: object




```python
data.iloc[33641]
```




    Message                 An account was successfully logged on.\r\n\r\n...
    Id                                                                   4624
    Version                                                               2.0
    Qualifiers                                                            NaN
    Level                                                                   0
    Task                                                                12544
    Opcode                                                                0.0
    Keywords                                             -9214364837600034816
    RecordId                                                              500
    ProviderName                          Microsoft-Windows-Security-Auditing
    ProviderId                           54849625-5478-4994-a5ba-3e3b0328c30d
    LogName                                                          Security
    ProcessId                                                           824.0
    ThreadId                                                            876.0
    MachineName                                               WIN-PTFPHFCRDJ0
    UserId                                                                NaN
    ActivityId                           eae887f7-a8b5-0000-7c8b-e8eab5a8d601
    RelatedActivityId                                                     NaN
    ContainerLog            c:\users\hyphens\hackthebox\event_horizon\proj...
    MatchedQueryIds                                           System.UInt32[]
    Bookmark                 System.Diagnostics.Eventing.Reader.EventBookmark
    LevelDisplayName                                              Information
    OpcodeDisplayName                                                    Info
    TaskDisplayName                                                     Logon
    KeywordsDisplayNames    System.Collections.ObjectModel.ReadOnlyCollect...
    Properties              System.Collections.Generic.List`1[System.Diagn...
    Date                                                           22/10/2020
    Time                                                             15:30:54
    Name: 33641, dtype: object




```python
data[data.Id == 4624].groupby("Date")["Time"].value_counts()
```




    Date        Time    
    22/10/2020  13:57:19    9
                13:57:20    6
                15:38:57    5
                13:52:08    4
                13:52:05    3
                           ..
    26/10/2020  19:08:16    1
                19:17:20    1
                19:23:13    1
                19:38:32    1
                19:39:05    1
    Name: Time, Length: 227, dtype: int64



## References

\[1\] Jurafsky D., Martin J. H. (2020). Speech and Language Processing An Introduction to Natural Language Processing, Computational Linguistics, and Speech Recognition. Retrieved from https://web.stanford.edu/~jurafsky/slp3/ on 3 June 2022. <br>

[2] Leskovec J., Rajaraman A., Ulman J. D. (2019). Mining of Massive Datasets. Retrieved from http://www.mmds.org/ on 3 June 2021.
