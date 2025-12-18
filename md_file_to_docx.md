# md_file_to_docx Function

```python
import json
import os
from typing import Union

def md_file_to_docx(md_file_path: str, docx_file_path: str) -> Union[str, dict]:
    """
    Convert a Markdown (.md) file to a Word (.docx) document.
    
    This function reads a Markdown file, parses its content, and converts it to
    a Microsoft Word document format while preserving basic formatting such as
    headers, bold text, italic text, lists, and paragraphs.
    
    Args:
        md_file_path (str): Path to the input Markdown file (.md)
        docx_file_path (str): Path where the output DOCX file will be saved
        
    Returns:
        Union[str, dict]: On success, returns JSON string with status and file path.
                         On error, returns dict with error details.
                         Success format: '{"status": "success", "file_path": "path/to/file.docx"}'
                         Error format: {"error": "Error message", "details": "Additional info"}
    """
    
    def _validate_parameters():
        """Validate input parameters."""
        if not isinstance(md_file_path, str) or not md_file_path.strip():
            raise ValueError("md_file_path must be a non-empty string")
        
        if not isinstance(docx_file_path, str) or not docx_file_path.strip():
            raise ValueError("docx_file_path must be a non-empty string")
        
        if not md_file_path.lower().endswith('.md'):
            raise ValueError("md_file_path must have .md extension")
        
        if not docx_file_path.lower().endswith('.docx'):
            raise ValueError("docx_file_path must have .docx extension")
        
        if not os.path.exists(md_file_path):
            raise FileNotFoundError(f"Markdown file not found: {md_file_path}")
        
        if not os.path.isfile(md_file_path):
            raise ValueError(f"Path is not a file: {md_file_path}")
    
    def _read_markdown_content():
        """Read and return the content of the markdown file."""
        try:
            with open(md_file_path, 'r', encoding='utf-8') as file:
                return file.read()
        except UnicodeDecodeError:
            with open(md_file_path, 'r', encoding='latin-1') as file:
                return file.read()
    
    def _create_output_directory():
        """Create output directory if it doesn't exist."""
        output_dir = os.path.dirname(os.path.abspath(docx_file_path))
        if output_dir and not os.path.exists(output_dir):
            os.makedirs(output_dir)
    
    def _parse_and_convert_markdown(content):
        """Parse markdown content and convert to DOCX."""
        from docx import Document
        from docx.shared import Pt
        from docx.enum.text import WD_PARAGRAPH_ALIGNMENT
        import re
        
        doc = Document()
        lines = content.split('\n')
        i = 0
        
        while i < len(lines):
            line = lines[i].strip()
            
            if not line:
                i += 1
                continue
            
            # Headers
            if line.startswith('#'):
                level = len(line) - len(line.lstrip('#'))
                text = line.lstrip('#').strip()
                heading = doc.add_heading(text, level=min(level, 9))
                
            # Lists
            elif line.startswith(('- ', '* ', '+ ')):
                list_items = []
                while i < len(lines) and lines[i].strip().startswith(('- ', '* ', '+ ')):
                    item_text = lines[i].strip()[2:].strip()
                    list_items.append(item_text)
                    i += 1
                i -= 1
                
                for item in list_items:
                    p = doc.add_paragraph()
                    p.style = 'List Bullet'
                    _add_formatted_text(p, item)
                    
            # Numbered lists
            elif re.match(r'^\d+\.', line):
                list_items = []
                while i < len(lines) and re.match(r'^\d+\.', lines[i].strip()):
                    item_text = re.sub(r'^\d+\.\s*', '', lines[i].strip())
                    list_items.append(item_text)
                    i += 1
                i -= 1
                
                for item in list_items:
                    p = doc.add_paragraph()
                    p.style = 'List Number'
                    _add_formatted_text(p, item)
                    
            # Regular paragraphs
            else:
                p = doc.add_paragraph()
                _add_formatted_text(p, line)
            
            i += 1
        
        return doc
    
    def _add_formatted_text(paragraph, text):
        """Add formatted text to a paragraph, handling bold and italic."""
        import re
        from docx.shared import Pt
        
        # Pattern to match **bold**, *italic*, and ***bold-italic***
        pattern = r'(\*{1,3})(.*?)\1'
        
        last_end = 0
        for match in re.finditer(pattern, text):
            # Add text before the match
            if match.start() > last_end:
                paragraph.add_run(text[last_end:match.start()])
            
            # Add formatted text
            markers = match.group(1)
            content = match.group(2)
            
            if len(markers) == 3:  # ***bold-italic***
                run = paragraph.add_run(content)
                run.bold = True
                run.italic = True
            elif len(markers) == 2:  # **bold**
                run = paragraph.add_run(content)
                run.bold = True
            else:  # *italic*
                run = paragraph.add_run(content)
                run.italic = True
            
            last_end = match.end()
        
        # Add remaining text
        if last_end < len(text):
            paragraph.add_run(text[last_end:])
    
    try:
        # Validate parameters
        _validate_parameters()
        
        # Read markdown content
        md_content = _read_markdown_content()
        
        # Create output directory if needed
        _create_output_directory()
        
        # Convert markdown to DOCX
        document = _parse_and_convert_markdown(md_content)
        
        # Save the document
        document.save(docx_file_path)
        
        # Return success response
        return json.dumps({
            "status": "success",
            "file_path": os.path.abspath(docx_file_path)
        })
        
    except ImportError as e:
        return {
            "error": "Missing required dependency",
            "details": f"Please install python-docx: pip install python-docx. Error: {str(e)}"
        }
    except FileNotFoundError as e:
        return {
            "error": "File not found",
            "details": str(e)
        }
    except PermissionError as e:
        return {
            "error": "Permission denied",
            "details": f"Cannot access file: {str(e)}"
        }
    except ValueError as e:
        return {
            "error": "Invalid parameter",
            "details": str(e)
        }
    except Exception as e:
        return {
            "error": "Conversion failed",
            "details": f"Unexpected error during conversion: {str(e)}"
        }
```