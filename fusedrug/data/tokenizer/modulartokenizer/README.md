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

* __init__() has two optional parameters: upper special token ID limit, and upper ID limit for all tokens. Depending on their values, there are three options of ID mapping:

        1. There is no limitation on IDs. In this case, new IDs are added after the last taken ID, and the ID space is
            compact, with no holes.

        2. There's an upper limit on all IDs (self._max_possible_token_id). In this case, the new IDs (regular and special)
            are also mapped after the last taken ID, and ID space is compact, but limited in size. Any tokens added beyond
            the limit must raise an exception.

        3. There's an upper limit in special IDs (self._max_special_token_id). In this case, special IDs are mapped after
            the last taken special ID and before special ID limit, and regular IDs are mapped after last taken regular ID
            (and before all ID limit, if such is defined). Any tokens mapped beyond the limit must raise an exception. The
            ID space consists of two parts (with an empty buffer between them):
            - [0..upper special ID limit], containing special IDs only, compacted at the beginning of the range
            - (upper special id limit, infinity or upper all ID limit], containing regular IDs only, compacted at the
                beginning of the range.

### add_special_tokens():
Adds a list of special tokens to the modular tokenizer. This does not change the existing tokenizer IDs, just adds new ones. If the modular tokenizer was created
with max_special_token_id, the special tokens will be mapped to IDs between max current special token ID and max_special_token_id.
### decode():
Decodes a list of tokens
### diagnose():
Tests a modular tokenizer for underlying ID mapping consistency, checking that the following hold for all sub-tokenizers:
* Special tokens are the same (and map to the same indices) across all the tokenizers
* Regular token ID mappings of any given tokenizer do not collide with special token mappings
* Regular token ID mappings of any given tokenizer do not collide with ID mappings of other tokenizers
### enable_padding():
Enables padding to a given length, using a given padding token. Also enables truncation of sequences to the same length.
### enable_truncation():
Enables truncation of encoded sequences to a given length. If padding is enabled, padding length is also set to given length.
### encode():
(Not implemented yet) Receives a string, infers which tokenizers to use on it and returns its tokenization.
### encode_list():
Receives a list of named tuples, each containing the type of tokenizer to use, a string to be tokenized, and, optionally, maximum length (in tokens) of the result. Tokenizes each input string.
### from_file():
Receives a path to a file or a directory and loads a modular tokenizer from that directory.
### get_added_vocab():
Returns a vocabulary of all special tokens (ones common between all subtokenizers)
### get_max_id():
Returns the highest mapped ID in the vocabulary, or the upper limit to ID, if it was set
### get_vocab_size():
Returns the size of the vocabulary of the modular tokenizer (i.e. the number of unique IDs, which may be greater than the number of unique tokens)
### id_to_token():
Returns the token that maps to the input ID.
### load():
Loads a modular tokenizer saved by save()
### load_from_jsons():
Loads a group of adjusted tokenizers (created by __init__, andsaved by save_jsons), and returns a modular tokenizer with the same ID mapping.
### save():
Saves all mudular tokenizer information to a given path.
### save_jsons():
Saves the underlying adjusted tokenizers as jsons.
### token_to_id():
Returns the input token's corresponding ID.
## Use example
### Creation:
An example of creation of a new modular tokenizer from a word-level AA sequence tokenizer and a BPE SMILES tokenizer is found here: [ModularTokenizer creation](test_multi_tokenizer_creation.py#L238)

It uses this config: [tokenizer_config.py](configs/tokenizer_config.yaml). Note: this line [path definition](configs/tokenizer_config.yaml#L3) needs to be changed so that _your_path_ points to cloned fuse-drug parent directory.

Additional tokens that are added to the newly created tokenizer are defined in [special_tokens.py](special_tokens.py) in special_tokens and task_tokens. Any additional tokens related to task definitions and queries need to be added to task_tokens. special_tokens and task_tokens are loaded in [ModularTokenizer creation](test_multi_tokenizer_creation.py#L248) by calling get_special_tokens_dict() and get_additional_tokens(["task"]), respectively.

#### General creation steps:
* Add all needed tokens to [special_tokens.py](special_tokens.py#L23)
* Collect all sub-tokenizer jsons, and add them to a config, similarly to [tokenizer_config.py](configs/tokenizer_config.yaml)
* Run [ModularTokenizer creation](test_multi_tokenizer_creation.py). The script will a. create a new tokenizer that contains all required added tokens and all the sub-tokenizers; and b. Test the resulting tokenizer for consistency.


### Usage:
An example of usage of the modular tokenizer is found here: [ModularTokenizer use](test_multi_tokenizer_use.py#L16). It uses the same config as the creation example, and loads the jsons that were saved by the creation code.

The example loads a ready ModularTokenizer stored here: [Word AA + BPE SMILES, path load](pretrained_tokenizers/modular_AA_SMILES_single_path) using load(), i.e. no specific json paths need to be specified, and the load config is automatically generated by save().

### Adding special tokens steps:
An example of adding special tokens to an existing tokenizer is found here: [ModularTokenizer update](test_multi_tokenizer_use.py#L35). The steps are as follows:
Short version:
* Update task_tokens in [special_tokens.py](special_tokens.py) with the required tokens in
* Run [ModularTokenizer use](test_multi_tokenizer_use.py)
Long version:
* Load an existing modular tokenizer, like [here](test_multi_tokenizer_use.py#L30)
* Create a list of the required tokens, similarly to [here](test_multi_tokenizer_use.py#L35)
* Call ModularTokenizer.add_special_tokens() with the list: [add_tokens](test_multi_tokenizer_use.py#L10)
* Save the new tokenizer, usually over the original modular tokenizer.

## Config structure:
The init and load_from_jsons functiona both receive a list of dictionaries, each defining a single type of tokenizer. The dictionaries have the following fields:
* name: Name of the tokenizer (usually depicting its use context - AA sequences, SMILES, etc)
* tokenizer_id:    unique int identifier of the tokenizer
* json_path:       a path to a json file containing the initial input tokenizer
* modular_json_path: a path to json that will contain the updated (remapped) sub-tokenizer that will be derived from the initial tokenizer (automatically generated by save())
* max_len: (Optional) maximum number of tokens encoded by each instance of this tokenizer. If not given or None - no limit is set. If max_len is defined both here and during a call to encode_list, the smallest one is used.
## Adding new tokens
There are two ways to add new tokens:
* Adding a whole new tokenizer, by calling ModularTokenizer.add_tokenizer
* Adding a list of special tokens (e.g. task-related), by calling ModularTokenizer.add_special_tokens()
