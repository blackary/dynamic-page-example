Here is some code demonstrating how you can make an app that pulls .py files dynamically from this repo and uses them as pages in the app 

```python
from datetime import timedelta
from pathlib import Path

import requests
import streamlit as st

pages_folder = Path("_pages")


@st.cache_data(show_spinner=False, ttl=300)
def list_python_files(owner: str, repo: str, ref: str = "main") -> list[dict]:
    """List all Python files in a GitHub repo using the Git Trees API."""
    tree_api = (
        f"https://api.github.com/repos/{owner}/{repo}/git/trees/{ref}?recursive=1"
    )
    payload = requests.get(tree_api).json()
    if "tree" not in payload:
        raise RuntimeError(f"Unexpected response: {payload}")
    files = []
    for item in payload["tree"]:
        if item.get("type") == "blob" and item.get("path", "").endswith(".py"):
            path = item["path"]
            raw_url = f"https://raw.githubusercontent.com/{owner}/{repo}/{ref}/{path}"
            files.append({"path": path, "raw_url": raw_url})
    return files


@st.cache_data(show_spinner=False, ttl=timedelta(hours=12))
def fetch_python_file_contents(files: list[dict]) -> list[Path]:
    """Fetch raw contents for each file entry returned by list_python_files."""
    # Create and clear pages folder
    pages_folder.mkdir(exist_ok=True)
    for file_path in pages_folder.iterdir():
        file_path.unlink()

    paths: list[Path] = []
    for file_entry in files:
        try:
            path = pages_folder / file_entry["path"]
            path.write_text(requests.get(file_entry["raw_url"]).text)
            paths.append(path)
        except requests.RequestException as exc:
            st.error(f"<error fetching file: {exc}>")
    return paths


file_contents = fetch_python_file_contents(
    list_python_files("blackary", "dynamic-page-example", "main")
)


pages = [st.Page(file_path) for file_path in file_contents]

current_page = st.navigation(pages)


st.sidebar.button("Re-fetch files", on_click=lambda: st.cache_data.clear())

current_page.run()
```

Current result: <img width="1255" height="500" alt="image" src="https://github.com/user-attachments/assets/aad825e4-dbb3-44ee-8cca-65d41b29efb0" />
