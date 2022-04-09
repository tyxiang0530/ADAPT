# ADAPT: Automated Demographic Analysis of Physicists in Textbooks
An automated tool for the identification of physicists and their demographic information in textbooks

## Introduction

Our intention is to analyze science textbooks used at Pomona and around the country, looking at representation of the scientists mentioned throughout. In order to do so, we have created a continually growing database of scientists in order to assist in the process. Data regarding race and ethnicity has been obtained through existing online sources, links for these online sources can be found below. Similarly, groupings of race and ethnicity have been done in accordance with the 2020 race and ethnicity categories also found below.

## How the Code Works

The tool consists of two classes: **NameGrab** and **WikiOperations**, as well as a third class **MainName** that simply runs the methods from the other two classes.

### NameGrab

#### Grabbing out a Name
 
(*If processing an index*) The first part is **NameGrab**, where the index is first separated by newline and each entry is placed into an array. We word tokenize these entries. All tokens are then tagged with named-entity recognition tags. We iterate through this array and remove all non-upper case tokens and a list of common stopwords. We complete all named-entity recognition prior to cleaning in order to preserve context and ensure accuracy of the named-entity recognition tagging process.
Since each entry index will typically only contain one name, we check each word token and build a full name out of the words. Names can be matched in two ways: if a token is tagged as a 'person' by named entity recognition, or if a token matches a name found in the physicist database (DB).

(*If processing a full book*) If an entire book is being processed, we follow the same process with some slight deviations. Unlike an index, there are no discrete entries that only contain a single idea. Everything is at the sentence level and there is higher complexity. We first sentence tokenize the input text into individual arrays and then split the sentence arrays with "ands" or commas into finer arrays. We know that each array now can maximally contain a single person. After performing these processing steps, we follow the same procedure as for the index to check if an array contains a person and build out their full name.

#### The Physicist Database and Additional Matching Details

 The physicist database is an extensive list of around 3000 unique dictionaries containing the following information: The name of the person, the name returned when wikipedia is queried, whether the person is a human, the gender of the person, the birth city of the person, the birth country of the person, the race of the person, a 'MatchName' variable that is used for matching in the **NameGrab** phase, a 'FullName' variable that is used for checking in the **WikiOperations** phase, the race of the person according to Wikipedia, and a 'MatchingNames' variable that is leftover for sanity checking purposes. We will discuss how the DB is constructed later.

 The variable that is used for name matching is the 'MatchName' variable. It contains only the last name of the physicist in the entry. We match last names instead of full names as we are iterating over tokens. This conveniently is also helpful to minimize the effect of OCR error, abbreviations, and odd formatting, as the simplicity reduces the chance of words being OCRed wrong.
 
If a token is a name, we add it to a running full name variable. After exhausting all the tokens in the array, we output the full name. Let us take a look at the example ['Erwin', 'Schrodinger']. We will match 'Erwin' and add it to FullName. Then we will match 'Schrodinger', and add it to the end of FullName. Now that the array is exhausted, we output 'Erwin Schrodinger' as the full name. This is the basic matching idea.
 
To reduce computation time, everytime we successfully match a token to the name in the database we also save the position we found the token in the database. Thus, the full output of the **NameGrab** class is the full name as well as its corresponding DB index position. If the entry is not in the DB, but it is tagged as “Person” by the NER, we only return the queried entry without an accompanying index.
 
There are multiple cases that can help simplify our job and increase accuracy. We will cover a few of the cases.
 
##### Case 1: An initial is present
 
We iterate through the tokens and check if the entry contains an initial (an uppercase letter followed by a period, ex: the “G.” in “G. Zweig”). We then check if there is a name present in the array. If there is an initial and a name, we know that the tokens in the array comprise a full name (This is a pattern we have found in all analyzed indices thus far). The full array is added to FullName and outputted.
 
##### Case 2: A dash is present
 
We then check if the entry contains a dash (ex: Biot-Savart). If it does, we assume each side of the dash contains a full name and analyze each part individually. If both sides are names, we give two outputs (ex: Biot-Savart will output both Biot and Savart).
 
At times, this dash splitting method can be problematic if an individual’s full last name consists of a dash (ex: Claude Cohen-Tannoudji will be analyzed as one individual named Cohen and another named Tannoudji). However, we still utilize this method as there is no way to discern between two names and a single name, and splitting of dashes achieves the highest accuracy.
 
It should also be noted that the last name matching method we use can lead to some confusion with names (Ex: “Taylor” can query both “Edwin Taylor” and “Brook Taylor” from the DB). This will be handled in the **WikiOperations** phase).
 
### Wiki Operations
 
In the **WikiOperations** phase, we look through each entry returned following **WikiOperations**. If the entry has a corresponding location in the DB, we first check if the database entry correctly corresponds to the tagged entry by checking if the first name and last name of the individual matches our database entry. If it does, we add a dictionary containing all of the info corresponding to the name from the database to a list. If the entries first and last name does not match or was not found by the DB and was tagged by NER, we query Wikipedia through the Wikimedia API and return a dictionary that is also added to the list. This dictionary entry is then added to our database so that it can be matched for any future processing. After all entries have been processed, we write the list of dictionaries to a csv. This is the final output of the tool.
 
## Sanitization
 
We check the "Matching Names" bool following the output of the csv. The “Matching Names” bool is True if the tagged entry matches the output entry, and False if otherwise. If True, the entry is left alone, but if marked as False, this indicates that the tagged entry differs from the queried entry. Usually, this results from OCR error, initial abbreviation mismatches (G. Zweig vs. George Zweig), full name mismatches (Edwin Taylor vs. Edwin F. Taylor) or accent mark mismatches (Rontgen vs. Roentgen). If that is the case, the entry is left alone. However, oftentimes if there is only a single name that is tagged, the incorrect entry may be queried (the tool may tag “Taylor” out of “Taylor Series” which should be attributed to “Brook Taylor”, however the automated querying may return “Edwin F. Taylor” instead) and this error must be manually corrected by requerying with the full name instead of only the partial.
 
## The Database
 
To build the database, we scrape lists of scientists and physicists from sources such as Wikipedia, (other sources) and add their names to a list. For each name, we query the Wikimedia API for the following information: “InstanceOf”, “Sex”, “Birthplace”. Each Wikipedia page typically contains information pertaining to each of these tags but if none are found, then the output is simply blank. If the “InstanceOf” tag for the name is not “Human (Q5)”, we remove the name from the list as we have found the Wikipedia page of something that is not a human. We add the information queried for each tag along with the name to our database.

Names that are found through NER during new textbook runs are also added to the database with the same process. This ensures that the database can expand as we run more and more books through it.
 
## Additional Information on Error
 
Much of the error in the automated process typically arises from the “pre-conditioning” step, in which, before the text is examined with our automated Python tool, the hard-copy textbook is scanned-in and the resulting pdf is subjected to optical character recognition. Distorted or blurry scans of hard-copy textbooks can lead to incorrect extraction of text. Common OCR errors arise from cases in which letters look similar, such as the lowercase “L” being confused for the uppercase “I”. 
 
 Errors due to OCR occur more commonly in the textbook chapter analyses because as text density increases, words are clustered closer together, increasing the likelihood of OCR confusion and incorrect text labeling. This can cause words to be split apart and trigger incorrect letter labeling. An increase in OCR error correlates to an increase in noise and thus a rise in false positives occurs, decreasing the precision score. For instance, a slight curvature in the spine of the book caused a distortion in the form of the word “for”, causing OCR to mark it as “Joi”. When put through the tool, “Joi” was tagged as a name and “Joi Ito” was output, generating a false positive. 
 
 This effect may also occur in the inverse case, where an alteration to a legitimate name due to OCR causes it to be missed by the software. Interestingly, in certain instances where letter substitution due to faulty OCR occurs, the tool is still able to recover information. One instance of faulty OCR occurred when the name “Daniel Bernoulli” was labeled as “Dcniel Brnulli”. However, as unknown names are queried through the Wikimedia API, slightly misspelled names can still query the correct entry. In this case, despite the misspelling, the correct “Daniel Bernoulli” information was still returned. OCR error is the central bottleneck to the effectiveness of the automated process in hand-scanned long-form texts.

