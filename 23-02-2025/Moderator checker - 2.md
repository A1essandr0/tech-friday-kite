Было:
https://gitlab.com/kite9/moderatorchecker/-/blob/f344c8bf3f0309bd44514e1638780052732c0a2a/app/request_processor.py
``` Python
async def process(args: Dict) -> Tuple[str, Dict, List[GroupFields]]:
    args_values_data = await redis_client.get_args_values(args['appMetrikaAPIKey'])
    
    group_fields = []
    group_data = None
    all_found_groups = {}
    
    for value_data_item in args_values_data:
        group_data = value_data_item
        matched_arguments_count = 0
        
        for field_name, values in value_data_item['fields'].items():
            if field_name not in args:
                group_fields = []
                group_data = None
                break
            if args[field_name].lower() not in values:
                group_fields = []
                group_data = None
                break

			matched_arguments_count += 1
            group_fields.append(GroupFields(name=field_name, value=args[field_name]))

		if group_data:
            if matched_arguments_count not in all_found_groups:
                all_found_groups[matched_arguments_count] = []
            all_found_groups[matched_arguments_count].append(group_data)

	if all_found_groups:
        group_data = all_found_groups[max(all_found_groups, key=int)][0]
        return group_data['name'], group_data['response'], group_fields

    return "moderator", await redis_client.get_moderator_response(args['appMetrikaAPIKey']), []
```

Проблемы с кодом - такие же:
- в функции есть сайд-эффекты - сложно тестировать
- большой объем нетривиальной и безымянной логики, вложенные циклы, куча условий
+ нужно добавить сюда еще один блок логики - куда и как это проще сделать, разобраться непросто


Стало:
https://gitlab.com/kite9/moderatorchecker/-/blob/5a953949af67c8f36148848d4f134c84edd6203d/app/group_checker.py
``` Python
def check_groups(internal_args: Dict[str, str], groups_records_list: List[Dict]) -> Tuple[str, Dict, List[GroupFields], bool]:
    all_found_groups = []
    
    for group_record in groups_records_list:
        check_result = check_group(internal_args, group_record)
        if check_result:
            all_found_groups.append(check_result)

    return select_best_match(all_found_groups)


def select_best_match(all_found_groups: List):
    if len(all_found_groups) > 0:
        all_found_groups.sort(key=lambda x: len(x["fields_matched"]))
        best_group_match = all_found_groups[-1]
        group_response, should_save_response = get_group_response(best_group_match)
        return (
            best_group_match['name'],
            group_response, 
            best_group_match['fields_matched'],
            should_save_response,
        )

    return ("moderator", None, [], False)


def check_group(internal_args: Dict[str, str], group_record: Dict) -> Dict:
    group_fields = []
    
    for field_name, possible_values in group_record['fields'].items():
        # Если в группе есть поле, которое не пришло в запросе, она не годится
        if field_name not in internal_args:
            return {}
        # Если значение поля из запроса не встречается в возможных значениях в группе, она не годится
        if internal_args[field_name].lower() not in possible_values:
            return {}
        group_fields.append(GroupFields(
            name=field_name, 
            value=internal_args[field_name]
        ))

    return {
        **group_record,
        "fields_matched": group_fields,
    }


def get_group_response(group_record: Dict) -> Tuple[Dict, bool]:
    """ Базовый ответ от группы имеет вес 100.
        Если есть альтернативные, то выбирается случайный из всех в соответствии с их весами.
    """
    base_response = group_record.get('response')
    alternative_responses = group_record.get('alternative_responses', [])
    if alternative_responses == []:
        return base_response, False
        
    base_response_weight = 100
    alternative_responses_weights = [
        item['weight']
        for item in alternative_responses
    ]
    
    responses = [base_response, *alternative_responses]
    weights = [base_response_weight, *alternative_responses_weights]
    result = choices(responses, weights=weights)[0]
    return result, True
```

- избавились от сайд-эффектов (в результате ВСЕ сайд-эффекты приложения собрались ровно в одном месте)
- разбили на функции, т.е. дали блокам логики внятные имена
- в результате можно отдельно тестировать каждый блок
- добавили дополнительную логику - просто как еще одну функцию