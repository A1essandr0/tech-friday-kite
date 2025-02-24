Было:
https://gitlab.com/kite9/moderatorchecker/-/blob/f344c8bf3f0309bd44514e1638780052732c0a2a/app/request_processor.py
``` Python
async def convert_synonyms_to_names(src_args: Dict) -> Dict:
    """
    Convert key value dict with synonyms to internal naming
    """
    args_synonyms_data = await redis_client.get_args_synonyms(src_args['p5'])
    
    if not args_synonyms_data:
        return {}
    
    args_synonyms, args = {}, {}
    for item in args_synonyms_data:
        args_synonyms[item['name']] = item['synonyms']
    
    synonyms_args = inverse_args_synonyms(args_synonyms)
    for key in src_args:
        internal_name = synonyms_args.get(key, key)
        args[internal_name] = src_args[key]
    return args


def inverse_args_synonyms(args_synonyms) -> Dict:
    """
    Inverse argument synonyms list to pairs synonym-argument
    """
    inversed_args_synonyms = {}
    for key in args_synonyms:
        for item in args_synonyms[key]:
            inversed_args_synonyms[item] = key

	return inversed_args_synonyms
```

Проблемы с кодом:
- 1я функция не чистая, в ней есть сайд-эффект обращения к Редису => юнит тестами функцию не покроешь (без трудоемких моков). А надо, т.к. логика выполняется нетривиальная.
- в логике с обилием вложенных словарей и списков уже вполне можно запутаться
- отсутствуют подходящие названия - не очень понятно, что здесь вообще происходит


Стало:
https://gitlab.com/kite9/moderatorchecker/-/blob/5a953949af67c8f36148848d4f134c84edd6203d/app/arguments_processor.py
``` Python
def process_arguments(input_args: Dict[str, str], synonyms_list: List[Dict]) -> Dict[str, str]:
    """
    Convert input arguments to internal naming
    """
    if not synonyms_list:
        return {}
        
    args_to_synonyms = unfold_mapping_list(synonyms_list)
    synonyms_to_args = invert_dict(args_to_synonyms)
    internal_args = make_internal_names(input_args, synonyms_to_args)
    return internal_args


def make_internal_names(input_args: Dict[str, str], synonyms: Dict) -> Dict[str, str]:
    """
    Transform input names/keys to internal based on dict of synonyms.
    Input values stay the same.
    """
    internal_args = {}
    for input_key, input_value in input_args.items():
        internal_name = synonyms.get(input_key, input_key)
        internal_args[internal_name] = input_value
    return internal_args


def unfold_mapping_list(mapping_list: List[Dict], key='name', value='synonyms') -> Dict:
    """
    Unfold list
    [ {key: 'Alpha', value: A}, {key: 'Beta', value: B}, ...] -> { 'Alpha': A, 'Beta': B, ...}
    """
    result = {}
    for item in mapping_list:
        result[item[key]] = item[value]
    return result


def invert_dict(src: Dict[str, List[str]]) -> Dict[str, str]:
    """
    Inverse dictionary
    {"A": ["a1", "a2"], ...} -> {"a1": "A", "a2": "A", ...}
    """
    inversed = {}
    for old_key, new_key_list in src.items():
        for new_key in new_key_list:
            inversed[new_key] = old_key
    return inversed
```

- сайд-эффектов больше нет, головная функция просто получает то, что ей нужно для процессинга, как параметры. Все функции чистые, и на них без проблем пишутся юниттесты.
- головная функция - пайплайн последовательной обработки входных данных. Каждый блок обработки выделен в маленькую функцию, и в результате безымянным блокам логики даны названия. В результате понятнее, что происходит.
- нейминг чуть-чуть получше
- тот случай, когда комментарии помогают