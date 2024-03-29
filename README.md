# scoresim
Open Source repository for the implementation of MusicScore Engine on Elasticsearch

###### ScoreSplit
Small program that splits scores JSON documents from NEUMA which separates signatures from each voice.
1 voice = 1 json document. 1 opus = several json documents (1 per voice)
Push the document into Elasticsearch, using the config file : data/config/config.json

The file "data/config/mapping.json" is used the first time to set the schema of json documents into the "scorelib" index stored in ES.

The source code for ScoreSim is in "src/fr/cnam/vertigo/scoresplit". A main function can be used to load files.

###### plugin configuration 
In order to specify your own ES scoring function, you must use a Java embedding program:
https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-scripting-engine.html

The source code for ScoreSim is in "src/fr/cnam/vertigo/scoresim"

To set the environment into ES:

- Create a Docker container "scoresim2" with Elasticsearch (and Kibana if possible). The image "nshou/elasticsearch-kibana" was used for testing (ES version 7.12.1). The container's name must be "scoresim2" (or change the script below)
- Copy the script "elasticsearch/installScoreSim_ES" into the container "/home/elasticsearch/elasticsearch-7.12.1/"
- Set the config file "elasticsearch/plugin-descriptor.properties" to the used elasticsearch version
	currently *7.12.1*

Import the ScoreSim2 Plugin:

- Generate a jar file containing binaries for "src/fr/cnam/vertigo/scoresim". The "scoresim.jardesc" can be used in Eclipse to generate it. The "scoresim.jar" must be put into "elasticsearch/"
- Produce a new ZIP archive 'scoresim.zip' *without* including the elasticsearch directory. To achieve this, the script "elasticsearch/installScoreSim_docker" is used. 
#|____elasticsearch/
#| |____   scoresim.jar <-- classes, resources, dependencies
#| |____   json-simple-1.1.jar
#| |____   plugin-descriptor.properties
- The previous script "installScoreSim_docker" executes automatically the script "installScoreSim_ES" from the container (copied previously). 
It will use: "bin/elasticsearch-plugin install file:///scoresim.zip"
- Restart the Elasticsearch server to make it work (usually, the container must be reset). 

###### SEARCH QUERIES ####

To test the scoresim function: 

POST /scorelib/_search
{
	"query": {
	  "function_score": {
	    "query": {
	      "bool": {
	        "must": [
	          {"wildcard": {"opusref": "*palestrina*"}},
	          {"match_phrase": {"diatonic": "5A;3D;2A;"}}
	        ]
	      }
	    },
	    "functions":
	      [{"script_score": {"script": {"lang": "ScoreSim", "source": "ScoreSim",
			    "params": {"query": "(0|1/2)(1|1)(2|1)", "type": 1}}}
	      }]
	  }
	},
	"highlight": {
    "fields" : {
      "diatonic" : {}, "summary": {}
    }
  },
  "fields":[
      "diatonic", "opusref", "summary"
  ],"_source": false
}

###### CHANGE THE SCORESIM PLUGIN

Several classes can be updated:

- fr.cnam.vertigo.scoresim.ScoreSimScriptPlugin
	Contains the ScriptEngine executed by ES.
	- Function "ScoreSimLeafFactory" is called when the query is initialized and generates a new "DistanceNeuma" object. It reads the parameters with the chosen query (param "query" with notes) and distance function (param "type", see "QuerySequence"). Change if a new param is given.
	- Function "newInstance" is generated for each matching json document. Nothing to change.
		the matching process is called by "ms.matchingPitches(stringPitches, distanceNeuma)"
- fr.cnam.vertigo.scoresim.distance.DistanceNeuma
	Encapsulates the QuerySequence generation (initialized with params "query" and "type"). It is used for each matching process of a "summary" string. 
	- "parseQuery()" generates the QuerySequence (using the "type")
		*** Change if the String value of the query is modified	
- fr.cnam.vertigo.scoresim.distance.QuerySequence
	Define all types of distance function that can be used and generates Objects accordingly (see in the following)
	Its an interface used for each distance function
	- the "public static final int SEQUENCE_xxx" defines the query types (used in ES query param "type")
	- the "public static QuerySequence getQuerySequence(int type, List<Pitch> pitches)" generates the Object "QuerySequence" with corresponding pitches (from the ES query param "query")
		*** Change to add your new scoring function and sequence matching.  
- fr.cnam.vertigo.scoresim.distance.xxx.QuerySequencexxx
	The implemented "QuerySequence" which gives all the matching queryblocks "MatchingSequence".
	- the "public MatchingSequence matchingSequence(String corpus, Pitch p)" see if the new pitch "p" generates a MatchingSequence according to the query (param) and previous pitches (previous calls of "matchingSequence")
		*** This function has to be changed in order to corresponds to the matching of sequences
	- the "initiateSequence()" must be called for each new iteration
	- the "endMatching()" must be called at the end of the iteration to take the last match if necessary.
	- the "distance(Sequence seq)" gives for a matching sequence the corresponding distance (type)
		*** This function has to be changed to give the score of a matching sequence.  
- fr.cnam.vertigo.scoresim.music.MusicSummary
	This class encapsulate the current JSON document from Elasticsearch on which we test MatchingSequences (by calling the QuerySequence in DistanceNeuma)
	- "private void getMatchingSequences (String corpus, String notes, DistanceNeuma distanceNeuma)" is called with field "summary" (containing notes' sequence) and iterates on each note and calls DistanceNeuma's QuerySequence.
		*** This function can be changed if the sequence of notes is modified 

To test your query locally, you can use "ScoreSimTestDistance" class.

### old features to update:

#Mapping to aggregate strings like "melody.value"
PUT /scorelib/opus_index/_mapping
{
  "properties": {
    "melody.value": {
      "type": "text",
      "fielddata": true
    }
  }
}

#Amount of scores per corpus
POST scorelib/_search
{"aggs" : { "nb_par_corpus" : {
"terms" : {"field" : "corpus_ref"} }}, "size":0}

#Amount of occurrences of a given pattern
POST scorelib/_search
{"aggs" : { "nb_par_corpus" : {
"terms" : {"field" : "melody.value"} }}, "size":0}
