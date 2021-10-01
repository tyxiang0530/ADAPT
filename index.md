# ADAPT: Automated Demographic Analysis of Physicists in Textbooks
#### An automated tool for the identification of physicists and their demographic information in textbooks

## Introduction

Our intention is to analyze science textbooks used at Pomona and around the country, looking at representation of the scientists mentioned throughout. In order to do so, we have created a continually growing database of scientists in order to assist in the process. Data regarding race and ethnicity has been obtained through existing online sources, links for these online sources can be found below. Similarly, groupings of race and ethnicity have been done in accordance with the 2020 race and ethnicity categories also found below.

## How the Code Works

The tool is comprised of two classes: <<NameGrab>> and <<WikiOperations>>. The first is <<NameGrab>>, where the index is first separated by newline and each entry is placed into an array. We word tokenize these entries. All tokens are then tagged with named-entity recognition tags. We iterate through this array and remove all non-upper case tokens and a list of common stopwords. We complete all named-entity recognition prior to cleaning in order to preserve context and ensure accuracy of the named-entity recognition tagging process.

Since each entry index will typically only contain one name, we check each word token and build a full name out of the words. Names can be matched in two ways: if a token is tagged as a 'person' by named entity recognition, or if a token matches a name found in the physicist database. If a token is a name, we add it to a running full name variable and after exhausting all the tokens in the array, we output the full name. Let us take a look at the example ['Erwin', 'Schrodinger']. We will match 'Erwin' and add it to FullName. Then we will match 'Schrodinger', and add it to the end of FullName. Now that the array is exhausted, we output 'Erwin Schrodinger' as the full name. This is the basic matching idea. However, there are multiple cases that can help simplify our job and increase accuracy. We will cover a few of the cases.
 
Case 1: An initial is present
  
We iterate through the tokens and check if the entry contains an initial (an uppercase letter followed by a period, ex: the “G.” in “G. Zweig”). We then check if there is a name present in the array. If there is an initial and a name, we know that the tokens in the array comprise a full name (This is a pattern we have found in all analyzed indices). The full array is added to FullName and outputted.
  
We then check if the entry contains a dash (ex: Biot-Savart). If it does, we assume each side of the dash contains a full name and analyze each part individually. We check if the last name of the tagged entry matches any entry in the database (this assists in reducing OCR error but may lead to confusion between individuals with the same last name. Ex: “Taylor” can query both “Edwin Taylor” and “Brook Taylor” from the DB. This will be handled in the <<WikiOperations phase). If the token is in the database, we set a name boolean to True, and note the position in the DB where it was found. We then return a dictionary containing the queried entry and its position in the database. If the entry is not in the DB, but it is tagged as “Person” by the NER, we only return the queried entry without an accompanying index.

In the <<WikiOperations>> phase, we look through each entry returned following <<NameGrab>>. If the entry has a corresponding location in the DB, we first check if the database entry correctly corresponds to the tagged entry by checking if the first name and last name of the individual matches our database entry. If it does, we add a dictionary containing the tagged name, the wiki name, the birth location, the birth country, the sex, the race, a bool describing whether the tagged name and wiki name match, the full name of the person, a truncated version of the name used for matching, and the wiki race to a list. If the entries first and last name does not match or was not found by the DB and was tagged by NER, we query Wikipedia through the Wikimedia API and return a dictionary that is also added to the list. This dictionary entry is then added to our database so that it can be matched for any future processing. After all entries have been processed, we write the list of dictionaries to a csv.

Sanitization
Following the output of a csv, the “Matching Names” bool is checked. The “Matching Names” bool is True if the tagged entry matches the output entry, and False if otherwise. If True, the entry is left alone, but if marked as False, this indicates the tagged entry differs from the queried entry. Usually, this results from OCR error, initial abbreviation mismatches (G. Zweig vs. George Zweig), full name mismatches (Edwin Taylor vs. Edwin F. Taylor) or accent mark mismatches (Rontgen vs. Roentgen). If that is the case, the entry is left alone. However, oftentimes if there is only a single name that is tagged, the incorrect entry may be queried (the tool may tag “Taylor” out of “Taylor Series” which should be attributed to “Brook Taylor”, however the automated querying may return “Edwin F. Taylor” instead) and this error must be manually corrected by requerying with the full name instead of only the partial. 





