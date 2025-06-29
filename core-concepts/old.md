
## Matching JSON Objects with an internal database

Structured generation will output JSON objects. To match these JSON objects with objects from an internal database, we recommend you to use the Levenshtein distance. The steps are the following: 
<Steps>
  <Step title="Normalize the values">
    Normalize the values of the JSON object by:
    - Flattening nested values
    - Removing all spacing 
    - Removing accents
    This makes it match the format in the internal database.
  </Step>
  <Step title="Compare using Levenshtein distance">
    Compare the normalized values using the Levenshtein distance algorithm to find matches.
  </Step>
</Steps>

Here is a python example:

```python
from typing import Any, List, Dict, Hashable
import re
import unicodedata
from Levenshtein import distance as levenshtein_distance
import pandas as pd
from rich.table import Table
from rich.console import Console

def normalize_value(val: Any) -> str:
    """Convert a value to uppercase and remove all spacing and accents for comparison."""
    if val is None:
        return ""
    # Convert to string, remove spacing and uppercase
    prep = re.sub(r'\s+', '', str(val).upper())
    # Remove accents (e.g. é -> E)
    return unicodedata.normalize('NFKD', prep).encode('ASCII', 'ignore').decode()

def levenshtein_similarity(val1: Any, val2: Any) -> float:
    """
    Calculate similarity between two values using the Levenshtein distance.
    Returns a similarity score between 0.0 and 1.0.
    """
    # If both values are "empty" or equal
    if (val1 or "") == (val2 or ""):
        return 1.0
    
    # For numeric comparisons, use a tolerance of 5%
    if isinstance(val1, (int, float)) and isinstance(val2, (int, float)):
        return 1.0 if abs(val1 - val2) <= 0.05 * max(abs(val1), abs(val2)) else 0.0

    # For non-numeric values, compare the normalized strings
    str1 = normalize_value(val1)
    str2 = normalize_value(val2)
    
    if str1 == str2:
        return 1.0
        
    if str1 and str2:
        max_len = max(len(str1), len(str2))
        if max_len == 0:
            return 1.0
        dist = levenshtein_distance(str1, str2)
        return 1 - (dist / max_len)
        
    return 0.0

from typing import TypedDict

class MatchResult(TypedDict):
    record: Any
    similarity: float


def find_top_k_neighbors(
    query: Dict[str, Any],
    database: List[Dict[Hashable, Any]], 
    k: int = 5
) -> List[MatchResult]:
    """
    Find the top k closest records in `database` to the `query` based on
    average Levenshtein similarity across all fields present in the query.

    Args:
        query: Dictionary containing the search criteria
        database: List of dictionaries containing the records to search
        k: Number of results to return

    Returns:
        List of dictionaries containing the k most similar records and their scores
    """
    compare_fields = list(query.keys())
    results: List[MatchResult] = []

    for record in database:
        score_sum = 0.0
        count = 0
        
        for field in compare_fields:
            val1 = query.get(field, "")
            val2 = record.get(field, "")
            sim = levenshtein_similarity(val1, val2)
            score_sum += sim
            count += 1

        if count > 0:
            avg_sim = score_sum / count
            results.append({
                "record": record,
                "similarity": avg_sim
            })

    # Sort by descending similarity and return the top k results
    results.sort(key=lambda x: x["similarity"], reverse=True)
    return results[:k]

def print_results_table(results: List[MatchResult]) -> None:
    """Print results in a formatted table"""
    console = Console()
    table = Table(title=f"Top {len(results)} Matches")

    # Get all unique fields from the records
    fields = set()
    for res in results:
        fields.update(res["record"].keys())
    fields = set(sorted(fields))

    # Add columns
    table.add_column("Similarity", justify="right", style="cyan")
    for field in fields:
        table.add_column(field, style="magenta")

    # Add rows
    for res in results:
        row = [f"{res['similarity']:.3f}"]
        for field in fields:
            row.append(str(res["record"].get(field, "")))
        table.add_row(*row)

    console.print(table)


def main()-> None:
    # Example usage:
    # Read CSV file
    df = pd.read_csv("data.csv")  # Replace with your CSV file path
    database = df.to_dict('records')

    # Example query using fields from your CSV
    query = {
        'Code Client': 'UIFOR001',
        'Nom': 'Retab',
        'Adresse 1': '5 Parvis Alan Turing',
        'Adresse 2': '',
        'CP': 75013,
        'Ville': 'PARIS',
        'Pays': 'FR',
        'Num TVA': 'FR00348236132',
        'SIRET': '32323219080032'
    }

    # Find matches
    results = find_top_k_neighbors(query, database, k=5)

    # Display results
    print_results_table(results)

```