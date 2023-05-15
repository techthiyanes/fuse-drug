# Modular Tokenizer:
* A modular tokenizer combines multiple pre-trained (huggingface-based) tokenizers and maps their tokens to a single, consistent ID space. It's useful for sequence-to-sequence problems, where different tokenizers should be used for different parts of the input sequence, depending on the context, and straightforward merging of the tokenizers may not be possible due to token overlap (e.g. 'C' in an amino acid sequence, standing for Cysteine, and 'C' in a SMILES sequence, standing for Carbon, should be mapped to different IDs). 
* The modular tokenizer retains most of huggingface tokenizer interface (but not the underlying logic), so it can be plugged into existing code with very few (if any at all) changes.
## Definitions:
* __sub-tokenizer__: One of several underlying tokenizers of the modular tokenizer. Maps to a consistent ID space.
* __initial (input) tokenizer__: One of several pre-trained tokenizers used to create a modular tokenizer. Maps to an ID space of its own, which may (and probably does) overlap with those of other initial tokenizers.
## Interface:
### __init__(): 
Creates a modular tokenizer that combines multiple initial tokenizers, adjusting them so that:
* They all share the same special tokens (combined special tokens from all the source tokenizers),
* Each tokenizer retains its regular tokens, however their IDs are remapped to a single space, with no overlaps.
Note: If a token has the same meaning across all input types (e.g. special tokens, like SEP, EOS, sentinel tokens), it should be defined as special token in at least one of the initial (input) tokenizers.
### save_jsons():
Saves the underlying adjusted tokenizers as jsons.
### load_from_jsons(): 
Loads a group of adjusted tokenizers (created by __init__, andsaved by save_jsons), and returns a modular tokenizer with the same ID mapping.
### diagnose():
Tests a modular tokenizer for underlying ID mapping consistency, checking that the following hold for all sub-tokenizers:
* Special tokens are the same (and map to the same indices) across all the tokenizers
* Regular token ID mappings of any given tokenizer do not collide with special token mappings
* Regular token ID mappings of any given tokenizer do not collide with ID mappings of other tokenizers
### encode_list():
Receives a list of named tuples, each containing the type of tokenizer to use, a string to be tokenized, and, optionally, maximum length (in tokens) of the result. Tokenizes each input string.
### encode(): 
(Not implemented yet) Receives a string, infers which tokenizers to use on it and returns its tokenization.
### decode():
Decodes a list of tokens
### get_vocab_size(): 
Returns the size of the vocabulary of the modular tokenizer (i.e. the number of unique IDs, which may be greater than the number of unique tokens)
### id_to_token():
Returns the token that maps to the input ID.
### token_to_id():
Returns the input token's corresponding ID.
## Use example
### Creation:
An example of creation of a new modular tokenizer from a word-level AA sequence tokenizer and a BPE SMILES tokenizer is found here: [ModularTokenizer creation](https://github.com/BiomedSciAI/fuse-drug/blob/a1b9564eb54b9fe39890645fb5378c13aedde6fb/fusedrug/data/tokenizer/modulartokenizer/test_multi_tokenizer_creation.py#L107)

It uses this config: [tokenizer_config.py](https://github.com/BiomedSciAI/fuse-drug/blob/main/fusedrug/data/tokenizer/modulartokenizer/configs/tokenizer_config.yaml). Note: this line [path definition](https://github.com/BiomedSciAI/fuse-drug/blob/a1b9564eb54b9fe39890645fb5378c13aedde6fb/fusedrug/data/tokenizer/modulartokenizer/configs/tokenizer_config.yaml#L3) needs to be changed so that _your_path_ points to cloned fuse-drug parent directory.
### Usage:
An example of usage of the modular tokenizer is found here: [ModularTokenizer use](https://github.com/BiomedSciAI/fuse-drug/blob/a1b9564eb54b9fe39890645fb5378c13aedde6fb/fusedrug/data/tokenizer/modulartokenizer/test_multi_tokenizer_use.py#L16). It uses the same config as the creation example, and loads the jsons that were saved by the creation code.

The example loads a ready ModularTokenizer stored here: [Word AA + BPE SMILES](https://github.com/BiomedSciAI/fuse-drug/tree/main/fusedrug/data/tokenizer/modulartokenizer/pretrained/modular_wordlevelAA_BPESMILES)
### Config structure:
The init and load_from_jsons functiona both receive a list of dictionaries, each defining a single type of tokenizer. The dictionaries have the following fields:
* name: Name of the tokenizer (usually depicting its use context - AA sequences, SMILES, etc)
* tokenizer_id:    unique int identifier of the tokenizer
* json_path:       a path to a json file containint the initial input tokenizer
* modular_json_path: a path to json that will contain the updated (remapped) sub-tokenizer that will be derived from the initial tokenizer:
* max_len: (Optional) maximum number of tokens encoded by each instance of this tokenizer. If not given or None - no limit is set. If max_len is defined both here and during a call to encode_list, the smallest one is used.