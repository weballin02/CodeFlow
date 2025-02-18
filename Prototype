import ast
import autopep8
import io
import zipfile
import streamlit as st

def auto_format(code):
    """
    Auto-format the given code using autopep8.
    """
    try:
        formatted = autopep8.fix_code(code)
        return formatted
    except Exception as e:
        st.error(f"Auto-formatting error: {e}")
        return code

def parse_script_to_blocks(script):
    """
    Parse a Python script and extract function definitions as code blocks.
    
    Args:
        script (str): Python code as a string.
    
    Returns:
        dict: Dictionary with function names as keys and their auto-formatted source code as values.
    """
    blocks = {}
    try:
        tree = ast.parse(script)
        # Iterate through top-level nodes.
        for node in tree.body:
            if isinstance(node, ast.FunctionDef):
                try:
                    # For Python 3.9+, use ast.unparse to get source code.
                    func_code = ast.unparse(node)
                except Exception:
                    func_code = f"def {node.name}(...):\n    ... # source unavailable"
                # Auto-format the code
                func_code = auto_format(func_code)
                blocks[node.name] = func_code
    except Exception as e:
        st.error(f"Error parsing script: {e}")
    return blocks

def extract_imports(blocks):
    """
    Extract import statements from a dictionary of code blocks.
    
    Args:
        blocks (dict): Dictionary of function blocks.
    
    Returns:
        list: List of unique import statement strings.
    """
    imports = []
    for code in blocks.values():
        try:
            tree = ast.parse(code)
            for node in tree.body:
                if isinstance(node, (ast.Import, ast.ImportFrom)):
                    try:
                        imp_str = ast.unparse(node)
                    except Exception:
                        imp_str = None
                    if imp_str:
                        imports.append(imp_str)
        except Exception:
            pass
    # Remove duplicates while preserving order.
    unique_imports = list(dict.fromkeys(imports))
    return unique_imports

def extract_docstrings(blocks):
    """
    Extract docstrings from each function block.
    
    Args:
        blocks (dict): Dictionary of function blocks.
    
    Returns:
        dict: Dictionary with function names as keys and docstrings (if any) as values.
    """
    docs = {}
    for name, code in blocks.items():
        try:
            tree = ast.parse(code)
            for node in tree.body:
                if isinstance(node, ast.FunctionDef):
                    doc = ast.get_docstring(node)
                    docs[name] = doc if doc else "No docstring provided."
        except Exception:
            docs[name] = "Error extracting docstring."
    return docs

def generate_main_file(blocks):
    """
    Generate a main entry point file content that includes:
      - Deduplicated import statements.
      - All function block definitions.
      - A main() function that calls each function.
    
    Args:
        blocks (dict): Dictionary of function blocks.
    
    Returns:
        str: The generated main file content.
    """
    imports = extract_imports(blocks)
    main_file_content = ""
    
    # Add import statements.
    if imports:
        main_file_content += "\n".join(imports) + "\n\n"
    
    # Add function block definitions.
    for code in blocks.values():
        main_file_content += auto_format(code) + "\n\n"
    
    # Add a main function that calls each function (customize execution order if needed).
    main_file_content += "def main():\n"
    for name in blocks.keys():
        main_file_content += f"    {name}()\n"
    main_file_content += "\n\nif __name__ == '__main__':\n    main()\n"
    
    # Auto-format the entire main file.
    return auto_format(main_file_content)

def create_zip_file(main_content, blocks):
    """
    Create a zip file in memory that contains the generated main file and a summary of blocks.
    
    Args:
        main_content (str): The content of the main file.
        blocks (dict): Dictionary of function blocks.
    
    Returns:
        BytesIO: In-memory zip file.
    """
    mem_zip = io.BytesIO()
    with zipfile.ZipFile(mem_zip, mode="w", compression=zipfile.ZIP_DEFLATED) as zf:
        zf.writestr("main.py", main_content)
        # Optionally include a file with individual blocks
        blocks_summary = ""
        for name, code in blocks.items():
            blocks_summary += f"### {name}\n{code}\n\n"
        zf.writestr("blocks_summary.txt", blocks_summary)
    mem_zip.seek(0)
    return mem_zip

# --- Initialize Session State ---
if 'blocks' not in st.session_state:
    st.session_state.blocks = {}

st.title("CodeFlow Manager - Enhanced Editor")
st.caption("Tooltips and inline help included for guidance.")

# --- Code Parsing Options ---
st.sidebar.header("Code Parsing Options")
st.sidebar.write("Upload or paste code to auto-parse function blocks.")

# Option 1: File Upload
st.sidebar.subheader("Upload Python Script")
uploaded_file = st.sidebar.file_uploader("Choose a .py file", type=["py"], help="Select a Python script to auto-parse functions.")
if uploaded_file is not None:
    script_content = uploaded_file.read().decode("utf-8")
    parsed_blocks = parse_script_to_blocks(script_content)
    if parsed_blocks:
        st.session_state.blocks.update(parsed_blocks)
        st.sidebar.success("Script parsed and function blocks added.")
    else:
        st.sidebar.error("No functions found in the uploaded script.")

# Option 2: Copy/Paste Code
st.sidebar.subheader("Or Paste Your Code")
pasted_code = st.sidebar.text_area("Paste your Python code here", height=150, help="Paste your code and click the button below to parse functions.")
if st.sidebar.button("Parse Pasted Code"):
    if pasted_code.strip():
        parsed_blocks = parse_script_to_blocks(pasted_code)
        if parsed_blocks:
            st.session_state.blocks.update(parsed_blocks)
            st.sidebar.success("Pasted code parsed and function blocks added.")
        else:
            st.sidebar.error("No functions found in the pasted code.")
    else:
        st.sidebar.error("No code provided.")

# --- Manual Block Management ---
st.sidebar.header("Manage Function Blocks")
action = st.sidebar.radio("Select Action", ["Create", "Edit", "Delete", "View All"], help="Choose an action to manage individual code blocks.")

if action == "Create":
    st.sidebar.subheader("Create New Block")
    new_block_name = st.sidebar.text_input("Block Name", value="function_new", help="Enter a unique function name.")
    new_code = st.sidebar.text_area("Function Code", value="def function_new():\n    \"\"\"Docstring for function_new.\"\"\"\n    pass", height=150, help="Enter the complete function code here.")
    if st.sidebar.button("Add Block"):
        if new_block_name in st.session_state.blocks:
            st.sidebar.error(f"Block '{new_block_name}' already exists!")
        else:
            st.session_state.blocks[new_block_name] = auto_format(new_code)
            st.sidebar.success(f"Block '{new_block_name}' added.")

elif action == "Edit":
    if st.session_state.blocks:
        st.sidebar.subheader("Edit Existing Block")
        block_to_edit = st.sidebar.selectbox("Select Block", list(st.session_state.blocks.keys()), help="Select a block to edit.")
        edited_code = st.sidebar.text_area("Edit Code", value=st.session_state.blocks[block_to_edit], height=150, help="Edit the function code and click update.")
        if st.sidebar.button("Update Block"):
            st.session_state.blocks[block_to_edit] = auto_format(edited_code)
            st.sidebar.success(f"Block '{block_to_edit}' updated.")
    else:
        st.sidebar.info("No blocks available to edit. Create or parse one first.")

elif action == "Delete":
    if st.session_state.blocks:
        st.sidebar.subheader("Delete Block")
        block_to_delete = st.sidebar.selectbox("Select Block", list(st.session_state.blocks.keys()), help="Select a block to delete.")
        if st.sidebar.button("Delete Block"):
            del st.session_state.blocks[block_to_delete]
            st.sidebar.success(f"Block '{block_to_delete}' deleted.")
    else:
        st.sidebar.info("No blocks available to delete.")

elif action == "View All":
    st.sidebar.subheader("All Function Blocks")
    if st.session_state.blocks:
        for name in st.session_state.blocks.keys():
            st.sidebar.markdown(f"**{name}**")
    else:
        st.sidebar.info("No blocks to view.")

# --- Main Display Area: Show Code Blocks ---
st.header("Current Function Blocks")
if st.session_state.blocks:
    for name, code in st.session_state.blocks.items():
        st.subheader(name)
        st.code(code, language="python")
else:
    st.info("No function blocks have been added yet.")

# --- Documentation Preview ---
st.header("Documentation Preview")
docstrings = extract_docstrings(st.session_state.blocks)
if docstrings:
    for name, doc in docstrings.items():
        with st.expander(f"Documentation for {name}"):
            st.write(doc)
else:
    st.info("No documentation available.")

# --- Generate the Main File with Auto-Imports ---
st.header("Generated Main File")
main_file_content = generate_main_file(st.session_state.blocks)
st.code(main_file_content, language="python")

# --- Export Project as ZIP ---
st.header("Export Project")
if st.button("Export Project as ZIP", help="Click to download a ZIP containing main.py and a summary of all function blocks."):
    zip_file = create_zip_file(main_file_content, st.session_state.blocks)
    st.download_button("Download ZIP", data=zip_file, file_name="codeflow_project.zip", mime="application/zip")
