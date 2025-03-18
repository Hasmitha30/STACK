import requests
import csv
from typing import List, Dict, Any

class PubMedFetcher:
    BASE_URL = "https://api.ncbi.nlm.nih.gov/lit/ctxp/v1/pubmed"

    def __init__(self, query: str):
        self.query = query

    def fetch_papers(self) -> List[Dict[str, Any]]:
        params = {"format": "json", "query": self.query}
        response = requests.get(self.BASE_URL, params=params)
        response.raise_for_status()
        return response.json().get("records", [])

    def filter_non_academic_authors(self, papers: List[Dict[str, Any]]) -> List[Dict[str, Any]]:
        filtered_papers = []
        for paper in papers:
            non_academic_authors = []
            company_affiliations = []
            for author in paper.get("authors", []):
                affiliation = author.get("affiliation", "").lower()
                if any(keyword in affiliation for keyword in ["pharma", "biotech", "company"]):
                    non_academic_authors.append(author.get("name"))
                    company_affiliations.append(affiliation)
            if non_academic_authors:
                paper["non_academic_authors"] = non_academic_authors
                paper["company_affiliations"] = company_affiliations
                filtered_papers.append(paper)
        return filtered_papers

    def save_to_csv(self, papers: List[Dict[str, Any]], filename: str):
        with open(filename, mode='w', newline='', encoding='utf-8') as file:
            writer = csv.writer(file)
            writer.writerow(["PubmedID", "Title", "Publication Date", "Non-academic Author(s)", "Company Affiliation(s)", "Corresponding Author Email"])
            for paper in papers:
                writer.writerow([
                    paper.get("uid"),
                    paper.get("title"),
                    paper.get("pubdate"),
                    "; ".join(paper.get("non_academic_authors", [])),
                    "; ".join(paper.get("company_affiliations", [])),
                    paper.get("corresponding_author_email", "")
                ])
import click
from pubmed_fetcher.fetcher import PubMedFetcher

@click.command()
@click.option('-q', '--query', required=True, help='Specify the query for fetching papers.')
@click.option('-f', '--file', default=None, help='Specify the filename to save the results. If not provided, prints to the console.')
@click.option('-d', '--debug', is_flag=True, help='Print debug information during execution.')
def get_papers_list(query, file, debug):
    if debug:
        click.echo(f"Fetching papers with query: {query}")
    
    fetcher = PubMedFetcher(query)
    papers = fetcher.fetch_papers()
    filtered_papers = fetcher.filter_non_academic_authors(papers)

    if file:
        fetcher.save_to_csv(filtered_papers, file)
        click.echo(f"Results saved to {file}")
    else:
        for paper in filtered_papers:
            click.echo(f"PubmedID: {paper.get('uid')}")
            click.echo(f"Title: {paper.get('title')}")
            click.echo(f"Publication Date: {paper.get('pubdate')}")
            click.echo(f"Non-academic Author(s): {', '.join(paper.get('non_academic_authors', []))}")
            click.echo(f"Company Affiliation(s): {', '.join(paper.get('company_affiliations', []))}")
            click.echo(f"Corresponding Author Email: {paper.get('corresponding_author_email', '')}")
            click.echo("-" * 80)

if __name__ == '__main__':
    get_papers_list()
