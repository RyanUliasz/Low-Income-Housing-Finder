## Overview

Demo for a GitHub repo for UF DCP4300 Project.


### Authors

Ryan Uliasz

### Example Input and Output 
| Input                      | Output                    |
|----------------------------|---------------------------|
|"What is the 60% AMI income cap for a 4-person family?" | $62,400   |
|"The rent limit for a 2-bedroom unit at 50% AMI?" | $1,170   |
|"What is the 2026 FMR for a 3-bedroom apartment?" | $1,868   |
|"Can you tell me the median family income for the county?" | $106,700   |
|"I need the 80 percent AMI income for a family of 5." | $89,856   |
|"What is the maximum rent at 120% AMI for a studio?" | $2,184   |


import re
from typing import Dict, List, Optional, Any
MEDIAN_INCOME: Dict[str, Any] = {
    'Alachua County': 106700
}

FAIR_MARKET_RENT: Dict[str, Any] = {
    '0 Bedroom FMR': 1154, '1 Bedroom FMR': 1246, '2 Bedroom FMR': 1493, 
    '3 Bedroom FMR': 1868, '4 Bedroom FMR': 1977
}

INCOME_LIMITS: List[Dict[str, Any]] = [
    {'ami': '30%', '1 Person': 21840, '2 Person': 24960, '3 Person': 28080, '4 Person': 31200, '5 Person': 33700, '6 Person': 36180},
    {'ami': '50%', '1 Person': 36400, '2 Person': 41600, '3 Person': 46800, '4 Person': 52000, '5 Person': 56160, '6 Person': 60320},
    {'ami': '60%', '1 Person': 43680, '2 Person': 49920, '3 Person': 56160, '4 Person': 62400, '5 Person': 67392, '6 Person': 72384},
    {'ami': '80%', '1 Person': 58240, '2 Person': 66560, '3 Person': 74880, '4 Person': 83200, '5 Person': 89856, '6 Person': 96512},
    {'ami': '120%', '1 Person': 87360, '2 Person': 99840, '3 Person': 112320, '4 Person': 124800, '5 Person': 134784, '6 Person': 144768},
]

RENT_LIMITS: List[Dict[str, Any]] = [
    {'ami': '30%', '0 Bedroom': 546, '1 Bedroom': 585, '2 Bedroom': 702, '3 Bedroom': 811, '4 Bedroom': 905},
    {'ami': '50%', '0 Bedroom': 910, '1 Bedroom': 975, '2 Bedroom': 1170, '3 Bedroom': 1352, '4 Bedroom': 1508},
    {'ami': '60%', '0 Bedroom': 1092, '1 Bedroom': 1170, '2 Bedroom': 1404, '3 Bedroom': 1623, '4 Bedroom': 1810},
    {'ami': '80%', '0 Bedroom': 1456, '1 Bedroom': 1560, '2 Bedroom': 1872, '3 Bedroom': 2164, '4 Bedroom': 2414},
    {'ami': '120%', '0 Bedroom': 2184, '1 Bedroom': 2340, '2 Bedroom': 2808, '3 Bedroom': 3246, '4 Bedroom': 3621},
]

def extract_entities(question: str) -> Dict[str, Optional[str]]:
    """
    Simulates the entity extraction component of the QA model using regex.
    This function now also checks for Median Income and FMR queries.
    """
    lower_question = question.lower()
    entities: Dict[str, Optional[str]] = {
        'type': None, # income, rent, fmr, median_income
        'ami': None,
        'size': None
    }

    # 1. Check for specific program/data types first (Median Income / FMR)
    if 'median' in lower_question and 'income' in lower_question:
        entities['type'] = 'median_income'
        return entities
    if 'fair market rent' in lower_question or 'fmr' in lower_question:
        entities['type'] = 'fmr'
        
    # 2. Check for AMI-based limits
    elif 'income' in lower_question or 'limit' in lower_question and 'rent' not in lower_question:
        entities['type'] = 'income'
    elif 'rent' in lower_question or 'bedroom' in lower_question:
        entities['type'] = 'rent'
    
    # Fallback to income if person/household is mentioned
    if entities['type'] is None and ('person' in lower_question or 'household' in lower_question):
        entities['type'] = 'income'

    # 3. Extract AMI Category (Only relevant for 'income' and 'rent' types)
    if entities['type'] in ['income', 'rent']:
        ami_match = re.search(r'(\d{2,3})%|(\d{2,3})\s*percent', lower_question)
        if ami_match:
            entities['ami'] = f"{ami_match.group(1) or ami_match.group(2)}%"

    # 4. Extract Size (Person or Bedroom)
    size_match = re.search(r'(\d)\s*-?(person|household|bed|bedroom)', lower_question)
    if size_match:
        number = size_match.group(1)
        entity_type = size_match.group(2)
        if 'person' in entity_type or 'household' in entity_type:
            entities['size'] = f"{number} Person"
        elif 'bed' in entity_type:
            # FMR size columns include " FMR" or " Limit", so we just need the number/type
            entities['size'] = f"{number} Bedroom"
    elif 'studio' in lower_question or '0-bed' in lower_question or '0 bed' in lower_question:
        entities['size'] = '0 Bedroom'
    
    return entities
def retrieve_factual_answer(question: str) -> str:
    """
    Takes a question, extracts key entities, and retrieves the corresponding
    factual limit from the internal data corpus across all four sheets.
    """
    entities = extract_entities(question)
    type_ = entities.get('type')
    ami = entities.get('ami')
    size = entities.get('size')
    
    formatted_value: str
    source: str
    year: str

    if type_ == 'median_income':
        value = MEDIAN_INCOME.get('Alachua County')
        formatted_value = f"${value:,.0f}"
        source = 'HUD Median Family Income (Sheet 2)'
        year = '2025'
        return f"The {year} {source} for Alachua County is: {formatted_value}"
    
    elif type_ == 'fmr':
        # FMR lookups need the size appended with ' FMR'
        fmr_key = f"{size} FMR" if size else "2 Bedroom FMR" # Default to 2 Bedroom FMR if size is missed
        value = FAIR_MARKET_RENT.get(fmr_key)
        
        if value is None:
            return f"Error: Could not find FMR data for the specified size ({size})."

        formatted_value = f"${value:,.0f}"
        source = 'HUD Fair Market Rent (Sheet 1)'
        year = '2026'
        return f"The {year} {source} for a {size} unit in Alachua County is: {formatted_value}"

    elif type_ in ['income', 'rent']:
        if not ami or not size:
            return f"Error: For {type_} limits, AMI percentage and size must be specified. Entities found: {entities}"
            
        data = INCOME_LIMITS if type_ == 'income' else RENT_LIMITS
        search_key = size

        # Search for the correct AMI row
        result_row: Optional[Dict[str, Any]] = next((row for row in data if row['ami'] == ami), None)

        if result_row:
            value = result_row.get(search_key)
            if value is not None:
                formatted_value = f"${value:,.0f}"
                source = 'Combined Income Limits (Sheet 3)' if type_ == 'income' else 'Combined Rent Limits (Sheet 4)'
                year = '2025'
                return (
                    f"The {year} {source} at the {ami} category for a {size} "
                    f"household/unit in Alachua County is: {formatted_value}"
                )
            else:
                return f"Data Not Found: Found the {ami} row, but no specific limit for a {size} in the data."
        else:
            return f"Data Not Found: The {ami} category is not listed in the {year} limits for Alachua County."

    else:
        return f"Error: Could not identify the type of housing data requested (Income, Rent, FMR, or Median Income)."
        if __name__ == '__main__':
    print("--- Low-Income Housing Factual Retrieval Simulator (Python) ---")
    print("Data Source: All four sheets of Alachua County Housing Limits (2025/2026)\n")

    test_questions = [
        # Sheet 3/4 Tests (AMI Limits)
        "What is the 60% AMI income limit for a 4-person household?",
        "What is the rent limit at 50 percent AMI for a 2-bedroom unit?",
        "What is the 120% AMI limit for 3 people?", 
        
        # Sheet 2 Test (Median Income)
        "What is the median family income for Alachua County?",
        
        # Sheet 1 Test (FMR)
        "What is the 2026 Fair Market Rent for a 3 bedroom unit?",
        "Tell me the FMR for a studio."
    ]

    for i, q in enumerate(test_questions):
        answer = retrieve_factual_answer(q)
        print(f"\n[{i+1}] Question: {q}")
        print(f"    Answer: {answer}")
        print("-" * 20)
