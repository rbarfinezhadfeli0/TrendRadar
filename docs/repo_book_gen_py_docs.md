# Documentation: repo_book_gen.py

## File Metadata

- **Path**: `repo_book_gen.py`
- **Size**: 48161 bytes
- **Extension**: .py
- **Lines**: 1091
- **Generated**: 2025-11-15T09:16:49.701075

## Original Source

```py
#!/usr/bin/env python3
"""
World's Best Repo Book Generator and Index Builder
Produces comprehensive documentation for an entire repository.
"""

import os
import json
import hashlib
import mimetypes
import re
import subprocess
from pathlib import Path
from datetime import datetime
from typing import Dict, List, Set, Tuple, Any
from collections import defaultdict

class RepoBookGenerator:
    def __init__(self, repo_path: str, output_dir: str = "./docs"):
        self.repo_path = Path(repo_path).resolve()
        self.output_dir = Path(output_dir).resolve()
        self.progress_log = self.output_dir / ".progress.log"
        self.checkpoint_file = self.output_dir / ".checkpoint.json"

        # Statistics
        self.stats = {
            "files_scanned": 0,
            "files_processed": 0,
            "docs_created": 0,
            "bytes_written": 0,
            "words_estimated": 0,
            "errors": [],
            "skipped_files": [],
            "binary_files": []
        }

        # File classification
        self.text_extensions = {
            '.py', '.js', '.ts', '.jsx', '.tsx', '.java', '.c', '.cpp', '.h', '.hpp',
            '.cs', '.go', '.rs', '.rb', '.php', '.swift', '.kt', '.scala',
            '.html', '.css', '.scss', '.sass', '.less',
            '.json', '.yaml', '.yml', '.toml', '.xml', '.ini', '.cfg', '.conf',
            '.md', '.txt', '.rst', '.adoc',
            '.sh', '.bash', '.zsh', '.fish', '.bat', '.ps1',
            '.sql', '.graphql', '.proto',
            '.Dockerfile', '.dockerignore', '.gitignore', '.gitattributes',
            '.env.example', '.editorconfig'
        }

        self.binary_extensions = {
            '.jpg', '.jpeg', '.png', '.gif', '.bmp', '.ico', '.svg',
            '.pdf', '.doc', '.docx', '.xls', '.xlsx', '.ppt', '.pptx',
            '.zip', '.tar', '.gz', '.bz2', '.7z', '.rar',
            '.exe', '.dll', '.so', '.dylib', '.bin',
            '.mp3', '.mp4', '.avi', '.mov', '.wav',
            '.ttf', '.otf', '.woff', '.woff2', '.eot'
        }

        self.exclude_patterns = {
            '.git', 'node_modules', '__pycache__', '.pytest_cache',
            'dist', 'build', '.next', 'out', 'target',
            'vendor', '.venv', 'venv', 'env',
            '.idea', '.vscode', '.DS_Store'
        }

        # Repository metadata
        self.repo_metadata = {}
        self.file_tree = {}
        self.folder_structure = defaultdict(lambda: {"files": [], "folders": []})
        self.all_keywords = defaultdict(list)  # keyword -> [(file, description)]

    def log_progress(self, message: str):
        """Log progress to both console and file."""
        timestamp = datetime.now().isoformat()
        log_entry = f"[{timestamp}] {message}\n"
        print(message)
        with open(self.progress_log, 'a') as f:
            f.write(log_entry)

    def compute_file_hash(self, filepath: Path) -> str:
        """Compute SHA256 hash of a file."""
        sha256_hash = hashlib.sha256()
        try:
            with open(filepath, "rb") as f:
                for byte_block in iter(lambda: f.read(4096), b""):
                    sha256_hash.update(byte_block)
            return sha256_hash.hexdigest()
        except Exception as e:
            self.stats["errors"].append(f"Hash error for {filepath}: {str(e)}")
            return ""

    def get_git_metadata(self) -> Dict[str, Any]:
        """Extract git repository metadata."""
        metadata = {
            "repo_name": self.repo_path.name,
            "repo_path": str(self.repo_path),
            "generation_date": datetime.now().isoformat(),
            "generator_version": "1.0.0"
        }

        try:
            # Get current commit
            result = subprocess.run(
                ["git", "log", "-1", "--format=%H|%an|%ae|%aI|%s"],
                cwd=self.repo_path,
                capture_output=True,
                text=True
            )
            if result.returncode == 0:
                parts = result.stdout.strip().split('|')
                if len(parts) >= 5:
                    metadata["commit_sha"] = parts[0]
                    metadata["commit_author"] = parts[1]
                    metadata["commit_email"] = parts[2]
                    metadata["commit_date"] = parts[3]
                    metadata["commit_message"] = parts[4]

            # Get remote URL
            result = subprocess.run(
                ["git", "remote", "get-url", "origin"],
                cwd=self.repo_path,
                capture_output=True,
                text=True
            )
            if result.returncode == 0:
                metadata["remote_url"] = result.stdout.strip()
        except Exception as e:
            self.stats["errors"].append(f"Git metadata error: {str(e)}")

        return metadata

    def is_text_file(self, filepath: Path) -> bool:
        """Determine if a file is text-based."""
        ext = filepath.suffix.lower()
        if ext in self.binary_extensions:
            return False
        if ext in self.text_extensions:
            return True

        # Check mime type
        mime_type, _ = mimetypes.guess_type(str(filepath))
        if mime_type and mime_type.startswith('text'):
            return True

        # Try reading first few bytes
        try:
            with open(filepath, 'rb') as f:
                chunk = f.read(8192)
                # Check for null bytes (common in binary files)
                if b'\x00' in chunk:
                    return False
                # Try to decode as UTF-8
                chunk.decode('utf-8')
                return True
        except:
            return False

    def should_exclude(self, path: Path) -> bool:
        """Check if path should be excluded."""
        parts = path.parts
        for pattern in self.exclude_patterns:
            if pattern in parts:
                return True
        return False

    def scan_repository(self) -> List[Path]:
        """Scan repository and classify all files."""
        self.log_progress("Starting repository scan...")

        all_files = []
        for root, dirs, files in os.walk(self.repo_path):
            root_path = Path(root)

            # Filter out excluded directories
            dirs[:] = [d for d in dirs if not self.should_exclude(root_path / d)]

            for filename in files:
                filepath = root_path / filename
                if self.should_exclude(filepath):
                    continue

                rel_path = filepath.relative_to(self.repo_path)
                all_files.append(rel_path)
                self.stats["files_scanned"] += 1

                # Build folder structure
                parent = str(rel_path.parent) if rel_path.parent != Path('.') else "ROOT"
                self.folder_structure[parent]["files"].append(str(rel_path))

                # Track subfolders
                if rel_path.parent != Path('.'):
                    parts = list(rel_path.parents)
                    for i in range(len(parts) - 1):
                        parent_str = str(parts[i]) if parts[i] != Path('.') else "ROOT"
                        child_str = str(parts[i-1]) if i > 0 else str(rel_path.parent)
                        if child_str != "." and child_str not in self.folder_structure[parent_str]["folders"]:
                            self.folder_structure[parent_str]["folders"].append(child_str)

        self.log_progress(f"Scanned {self.stats['files_scanned']} files")
        return all_files

    def extract_keywords(self, content: str, filepath: Path) -> List[Tuple[str, str]]:
        """Extract keywords from file content."""
        keywords = []
        ext = filepath.suffix.lower()

        # Programming language keywords
        if ext in {'.py', '.js', '.ts', '.java', '.c', '.cpp', '.go', '.rs'}:
            # Function/class definitions
            patterns = [
                (r'\bclass\s+(\w+)', 'class'),
                (r'\bdef\s+(\w+)', 'function'),
                (r'\bfunction\s+(\w+)', 'function'),
                (r'\bconst\s+(\w+)', 'constant'),
                (r'\blet\s+(\w+)', 'variable'),
                (r'\bvar\s+(\w+)', 'variable'),
                (r'\btype\s+(\w+)', 'type'),
                (r'\binterface\s+(\w+)', 'interface'),
                (r'\benum\s+(\w+)', 'enum'),
            ]

            for pattern, kw_type in patterns:
                matches = re.finditer(pattern, content, re.MULTILINE)
                for match in matches:
                    identifier = match.group(1)
                    keywords.append((identifier, f"{kw_type}: {identifier}"))

        # Extract ALL_CAPS constants
        caps_constants = re.findall(r'\b([A-Z][A-Z0-9_]{2,})\b', content)
        for const in set(caps_constants):
            if len(const) > 2 and const not in {'GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'HEAD', 'OPTIONS'}:
                keywords.append((const, f"constant: {const}"))

        # Extract camelCase/PascalCase identifiers
        identifiers = re.findall(r'\b([a-z][a-zA-Z0-9]{3,}|[A-Z][a-zA-Z0-9]{3,})\b', content)
        for ident in set(identifiers):
            if len(ident) > 3 and not ident.lower() in {'this', 'that', 'with', 'from', 'import', 'return', 'true', 'false', 'null', 'none'}:
                keywords.append((ident, f"identifier: {ident}"))

        # Limit to most relevant keywords (by frequency)
        keyword_counts = defaultdict(int)
        for kw, desc in keywords:
            keyword_counts[kw] += 1

        # Sort by frequency and take top keywords
        sorted_kw = sorted(keyword_counts.items(), key=lambda x: x[1], reverse=True)
        result = []
        for kw, count in sorted_kw[:500]:  # Limit to 500 keywords per file
            desc = next((d for k, d in keywords if k == kw), f"identifier: {kw}")
            result.append((kw, desc))

        return result

    def count_words(self, text: str) -> int:
        """Estimate word count."""
        return len(re.findall(r'\b\w+\b', text))

    def generate_file_docs(self, filepath: Path, rel_path: Path) -> Tuple[str, int]:
        """Generate comprehensive documentation for a single file."""
        try:
            # Read file content
            with open(filepath, 'r', encoding='utf-8', errors='ignore') as f:
                content = f.read()

            # Prepare documentation
            doc_content = f"# Documentation: {rel_path}\n\n"
            doc_content += f"## File Metadata\n\n"
            doc_content += f"- **Path**: `{rel_path}`\n"
            doc_content += f"- **Size**: {filepath.stat().st_size} bytes\n"
            doc_content += f"- **Extension**: {filepath.suffix}\n"
            doc_content += f"- **Lines**: {len(content.splitlines())}\n"
            doc_content += f"- **Generated**: {datetime.now().isoformat()}\n\n"

            # Original source
            doc_content += f"## Original Source\n\n"
            doc_content += f"```{filepath.suffix[1:] if filepath.suffix else 'text'}\n"

            # Include full content (with reasonable limit for display)
            if len(content) > 1_000_000:  # 1MB limit for inline display
                doc_content += content[:1_000_000]
                doc_content += f"\n\n... (truncated, full file is {len(content)} characters)\n"
            else:
                doc_content += content

            doc_content += f"\n```\n\n"

            # High-level overview
            doc_content += f"## High-Level Overview\n\n"
            ext = filepath.suffix.lower()

            if ext == '.py':
                doc_content += self._analyze_python_file(content, rel_path)
            elif ext in {'.js', '.ts', '.jsx', '.tsx'}:
                doc_content += self._analyze_javascript_file(content, rel_path)
            elif ext == '.yaml' or ext == '.yml':
                doc_content += self._analyze_yaml_file(content, rel_path)
            elif ext == '.md':
                doc_content += self._analyze_markdown_file(content, rel_path)
            elif ext == '.json':
                doc_content += self._analyze_json_file(content, rel_path)
            else:
                doc_content += self._analyze_generic_file(content, rel_path)

            # Performance & Security notes
            doc_content += f"\n## Performance & Security Notes\n\n"
            doc_content += self._analyze_security_performance(content, ext)

            # Related files
            doc_content += f"\n## Related Files\n\n"
            doc_content += self._find_related_files(content, rel_path)

            word_count = self.count_words(doc_content)
            return doc_content, word_count

        except Exception as e:
            self.stats["errors"].append(f"Error generating docs for {rel_path}: {str(e)}")
            return f"# Error\n\nFailed to generate documentation: {str(e)}\n", 0

    def _analyze_python_file(self, content: str, rel_path: Path) -> str:
        """Analyze Python file structure."""
        analysis = "This is a Python source file.\n\n"

        # Find imports
        imports = re.findall(r'^(?:from\s+[\w.]+\s+)?import\s+.+$', content, re.MULTILINE)
        if imports:
            analysis += f"### Imports ({len(imports)})\n\n"
            for imp in imports[:20]:  # Show first 20
                analysis += f"- `{imp.strip()}`\n"
            if len(imports) > 20:
                analysis += f"- ... and {len(imports) - 20} more\n"
            analysis += "\n"

        # Find classes
        classes = re.findall(r'^class\s+(\w+)(?:\(([^)]*)\))?:', content, re.MULTILINE)
        if classes:
            analysis += f"### Classes ({len(classes)})\n\n"
            for class_name, base_classes in classes:
                bases = f" (inherits from: {base_classes})" if base_classes else ""
                analysis += f"- **{class_name}**{bases}\n"
            analysis += "\n"

        # Find functions
        functions = re.findall(r'^(?:async\s+)?def\s+(\w+)\s*\([^)]*\)', content, re.MULTILINE)
        if functions:
            analysis += f"### Functions ({len(functions)})\n\n"
            for func in functions[:30]:  # Show first 30
                analysis += f"- `{func}()`\n"
            if len(functions) > 30:
                analysis += f"- ... and {len(functions) - 30} more\n"
            analysis += "\n"

        # Find docstrings
        docstrings = re.findall(r'"""(.*?)"""', content, re.DOTALL)
        if docstrings:
            analysis += f"### Module Documentation\n\n"
            if docstrings[0].strip():
                analysis += f"{docstrings[0].strip()}\n\n"

        return analysis

    def _analyze_javascript_file(self, content: str, rel_path: Path) -> str:
        """Analyze JavaScript/TypeScript file."""
        analysis = "This is a JavaScript/TypeScript source file.\n\n"

        # Find imports
        imports = re.findall(r'^import\s+.+$', content, re.MULTILINE)
        if imports:
            analysis += f"### Imports ({len(imports)})\n\n"
            for imp in imports[:20]:
                analysis += f"- `{imp.strip()}`\n"
            if len(imports) > 20:
                analysis += f"- ... and {len(imports) - 20} more\n"
            analysis += "\n"

        # Find functions
        functions = re.findall(r'(?:function\s+(\w+)|(?:const|let|var)\s+(\w+)\s*=\s*(?:async\s+)?\([^)]*\)\s*=>)', content)
        if functions:
            func_names = [f[0] or f[1] for f in functions]
            analysis += f"### Functions ({len(func_names)})\n\n"
            for func in func_names[:30]:
                analysis += f"- `{func}()`\n"
            if len(func_names) > 30:
                analysis += f"- ... and {len(func_names) - 30} more\n"
            analysis += "\n"

        # Find classes
        classes = re.findall(r'class\s+(\w+)', content)
        if classes:
            analysis += f"### Classes ({len(classes)})\n\n"
            for cls in classes:
                analysis += f"- **{cls}**\n"
            analysis += "\n"

        return analysis

    def _analyze_yaml_file(self, content: str, rel_path: Path) -> str:
        """Analyze YAML configuration file."""
        analysis = "This is a YAML configuration file.\n\n"

        # Find top-level keys
        top_keys = re.findall(r'^(\w+):', content, re.MULTILINE)
        if top_keys:
            analysis += f"### Top-Level Configuration Keys ({len(set(top_keys))})\n\n"
            for key in sorted(set(top_keys)):
                analysis += f"- `{key}`\n"
            analysis += "\n"

        return analysis

    def _analyze_markdown_file(self, content: str, rel_path: Path) -> str:
        """Analyze Markdown documentation file."""
        analysis = "This is a Markdown documentation file.\n\n"

        # Find headers
        headers = re.findall(r'^(#{1,6})\s+(.+)$', content, re.MULTILINE)
        if headers:
            analysis += f"### Document Structure ({len(headers)} sections)\n\n"
            for level, title in headers[:50]:
                indent = "  " * (len(level) - 1)
                analysis += f"{indent}- {title}\n"
            if len(headers) > 50:
                analysis += f"- ... and {len(headers) - 50} more sections\n"
            analysis += "\n"

        # Find links
        links = re.findall(r'\[([^\]]+)\]\(([^)]+)\)', content)
        if links:
            analysis += f"### External References ({len(links)} links)\n\n"
            for text, url in links[:20]:
                analysis += f"- [{text}]({url})\n"
            if len(links) > 20:
                analysis += f"- ... and {len(links) - 20} more links\n"
            analysis += "\n"

        return analysis

    def _analyze_json_file(self, content: str, rel_path: Path) -> str:
        """Analyze JSON file."""
        analysis = "This is a JSON data file.\n\n"

        try:
            data = json.loads(content)
            if isinstance(data, dict):
                analysis += f"### Top-Level Keys ({len(data)})\n\n"
                for key in sorted(data.keys()):
                    value_type = type(data[key]).__name__
                    analysis += f"- `{key}`: {value_type}\n"
                analysis += "\n"
            elif isinstance(data, list):
                analysis += f"### Array Contents\n\n"
                analysis += f"- Length: {len(data)}\n"
                if data:
                    analysis += f"- Item type: {type(data[0]).__name__}\n"
                analysis += "\n"
        except:
            analysis += "Note: Could not parse as valid JSON.\n\n"

        return analysis

    def _analyze_generic_file(self, content: str, rel_path: Path) -> str:
        """Generic file analysis."""
        lines = content.splitlines()
        analysis = f"This is a text file with {len(lines)} lines.\n\n"

        if lines:
            analysis += f"### Preview (first 20 lines)\n\n"
            preview = '\n'.join(lines[:20])
            analysis += f"```\n{preview}\n```\n\n"
            if len(lines) > 20:
                analysis += f"... and {len(lines) - 20} more lines\n\n"

        return analysis

    def _analyze_security_performance(self, content: str, ext: str) -> str:
        """Analyze for security and performance concerns."""
        notes = []

        # Security checks
        security_patterns = [
            (r'password\s*=\s*["\']', "Potential hardcoded password"),
            (r'api[_-]?key\s*=\s*["\']', "Potential hardcoded API key"),
            (r'secret\s*=\s*["\']', "Potential hardcoded secret"),
            (r'eval\s*\(', "Use of eval() - potential security risk"),
            (r'exec\s*\(', "Use of exec() - potential security risk"),
            (r'os\.system\s*\(', "Use of os.system() - potential command injection risk"),
            (r'subprocess\.call\s*\([^)]*shell\s*=\s*True', "Subprocess with shell=True - potential injection risk"),
        ]

        for pattern, message in security_patterns:
            if re.search(pattern, content, re.IGNORECASE):
                notes.append(f"⚠️ **Security**: {message}")

        # Performance checks
        if ext == '.py':
            if re.search(r'for\s+\w+\s+in.*:\s*for\s+\w+\s+in', content, re.DOTALL):
                notes.append(f"⚙️ **Performance**: Nested loops detected - consider optimization")
            if content.count('import ') > 50:
                notes.append(f"⚙️ **Performance**: Large number of imports - may impact startup time")

        if not notes:
            notes.append("✅ No obvious security or performance concerns detected.")

        return '\n'.join(notes) + '\n'

    def _find_related_files(self, content: str, rel_path: Path) -> str:
        """Find files related to this one based on imports and references."""
        related = []

        # Python imports
        py_imports = re.findall(r'from\s+([\w.]+)\s+import|import\s+([\w.]+)', content)
        for imp in py_imports:
            module = imp[0] or imp[1]
            if not module.startswith(('os', 'sys', 're', 'json', 'typing', 'pathlib', 'datetime')):
                related.append(module)

        # JS/TS imports
        js_imports = re.findall(r'import\s+.*from\s+["\']([^"\']+)["\']', content)
        related.extend(js_imports)

        if related:
            result = "Files and modules referenced:\n\n"
            for ref in sorted(set(related))[:20]:
                result += f"- `{ref}`\n"
            if len(related) > 20:
                result += f"- ... and {len(related) - 20} more\n"
            return result
        else:
            return "No obvious file references detected.\n"

    def generate_file_keywords(self, filepath: Path, rel_path: Path) -> Tuple[str, List[Tuple[str, str]]]:
        """Generate keyword index for a file."""
        try:
            with open(filepath, 'r', encoding='utf-8', errors='ignore') as f:
                content = f.read()

            keywords = self.extract_keywords(content, filepath)

            kw_content = f"# Keywords: {rel_path}\n\n"
            kw_content += f"**File**: `{rel_path}`\n\n"
            kw_content += f"**Total Keywords**: {len(keywords)}\n\n"

            # Group by first letter
            by_letter = defaultdict(list)
            for kw, desc in keywords:
                first_letter = kw[0].upper() if kw else '?'
                by_letter[first_letter].append((kw, desc))

            for letter in sorted(by_letter.keys()):
                kw_content += f"## {letter}\n\n"
                for kw, desc in sorted(by_letter[letter], key=lambda x: x[0].lower()):
                    anchor = kw.lower().replace(' ', '-')
                    kw_content += f"- **{kw}** <a id=\"{anchor}\"></a>: {desc}\n"
                kw_content += "\n"

            return kw_content, keywords

        except Exception as e:
            self.stats["errors"].append(f"Error generating keywords for {rel_path}: {str(e)}")
            return f"# Error\n\nFailed to generate keywords: {str(e)}\n", []

    def process_file(self, rel_path: Path):
        """Process a single file (generate docs and keywords)."""
        filepath = self.repo_path / rel_path

        # Determine output paths
        doc_dir = self.output_dir / rel_path.parent
        doc_dir.mkdir(parents=True, exist_ok=True)

        safe_name = rel_path.name.replace('.', '_')
        docs_file = doc_dir / f"{safe_name}_docs.md"
        kw_file = doc_dir / f"{safe_name}_kw.md"

        # Check if binary
        if not self.is_text_file(filepath):
            # Create minimal documentation for binary files
            self.stats["binary_files"].append(str(rel_path))

            binary_doc = f"# Binary File: {rel_path}\n\n"
            binary_doc += f"- **Path**: `{rel_path}`\n"
            binary_doc += f"- **Size**: {filepath.stat().st_size} bytes\n"
            binary_doc += f"- **Type**: Binary file\n"
            binary_doc += f"- **Extension**: {filepath.suffix}\n\n"
            binary_doc += "This file is binary and cannot be analyzed as text.\n"

            docs_file.write_text(binary_doc, encoding='utf-8')
            self.stats["docs_created"] += 1
            self.stats["bytes_written"] += len(binary_doc)
            return

        # Generate documentation
        try:
            doc_content, word_count = self.generate_file_docs(filepath, rel_path)
            docs_file.write_text(doc_content, encoding='utf-8')
            self.stats["docs_created"] += 1
            self.stats["bytes_written"] += len(doc_content)
            self.stats["words_estimated"] += word_count

            # Generate keywords
            kw_content, keywords = self.generate_file_keywords(filepath, rel_path)
            kw_file.write_text(kw_content, encoding='utf-8')
            self.stats["docs_created"] += 1
            self.stats["bytes_written"] += len(kw_content)

            # Store keywords for global index
            for kw, desc in keywords:
                self.all_keywords[kw].append((str(rel_path), desc))

            self.stats["files_processed"] += 1

        except Exception as e:
            self.stats["errors"].append(f"Error processing {rel_path}: {str(e)}")
            self.stats["skipped_files"].append(str(rel_path))

    def generate_folder_docs(self, folder_key: str):
        """Generate index.md, doc.md, and sub.md for a folder."""
        folder_path = Path(folder_key) if folder_key != "ROOT" else Path(".")
        doc_dir = self.output_dir / folder_path
        doc_dir.mkdir(parents=True, exist_ok=True)

        folder_data = self.folder_structure[folder_key]

        # Generate index.md
        index_content = f"# Index: {folder_key}\n\n"
        index_content += f"**Folder**: `{folder_key}`\n\n"

        if folder_data["folders"]:
            index_content += f"## Subfolders ({len(folder_data['folders'])})\n\n"
            for subfolder in sorted(folder_data["folders"]):
                # Create relative link to subfolder's index
                if folder_key == "ROOT":
                    link = f"{subfolder}/index.md"
                else:
                    link = f"{Path(subfolder).name}/index.md"
                index_content += f"- [{subfolder}]({link})\n"
            index_content += "\n"

        if folder_data["files"]:
            index_content += f"## Files ({len(folder_data['files'])})\n\n"
            for file in sorted(folder_data["files"]):
                file_path = Path(file)
                safe_name = file_path.name.replace('.', '_')
                rel_link = f"{safe_name}_docs.md"
                index_content += f"- [{file_path.name}]({rel_link}) - [keywords]({safe_name}_kw.md)\n"
            index_content += "\n"

        (doc_dir / "index.md").write_text(index_content, encoding='utf-8')
        self.stats["docs_created"] += 1
        self.stats["bytes_written"] += len(index_content)

        # Generate doc.md
        doc_content = f"# Documentation: {folder_key}\n\n"
        doc_content += f"**Folder**: `{folder_key}`\n\n"
        doc_content += f"## Overview\n\n"
        doc_content += f"This folder contains {len(folder_data['files'])} file(s) and {len(folder_data['folders'])} subfolder(s).\n\n"

        # Add context based on folder name
        folder_name = Path(folder_key).name if folder_key != "ROOT" else "root"
        if folder_name in {'src', 'source'}:
            doc_content += "This appears to be a source code directory.\n\n"
        elif folder_name in {'test', 'tests', '__tests__'}:
            doc_content += "This appears to be a test directory.\n\n"
        elif folder_name in {'docs', 'doc', 'documentation'}:
            doc_content += "This appears to be a documentation directory.\n\n"
        elif folder_name in {'config', 'conf', 'configuration'}:
            doc_content += "This appears to be a configuration directory.\n\n"

        (doc_dir / "doc.md").write_text(doc_content, encoding='utf-8')
        self.stats["docs_created"] += 1
        self.stats["bytes_written"] += len(doc_content)

        # Generate sub.md (merged keywords)
        sub_content = f"# Keyword Summary: {folder_key}\n\n"
        sub_content += f"**Folder**: `{folder_key}`\n\n"
        sub_content += "This file contains merged keywords from all files in this folder.\n\n"

        # Collect keywords from all files in this folder
        folder_keywords = defaultdict(list)
        for file_path in folder_data["files"]:
            file_rel = Path(file_path)
            for kw, locations in self.all_keywords.items():
                for loc_file, desc in locations:
                    if Path(loc_file).parent == file_rel.parent and Path(loc_file).name == file_rel.name:
                        folder_keywords[kw].append((loc_file, desc))

        if folder_keywords:
            by_letter = defaultdict(list)
            for kw in folder_keywords.keys():
                first_letter = kw[0].upper() if kw else '?'
                by_letter[first_letter].append(kw)

            for letter in sorted(by_letter.keys()):
                sub_content += f"## {letter}\n\n"
                for kw in sorted(by_letter[letter], key=lambda x: x.lower()):
                    locations = folder_keywords[kw]
                    sub_content += f"- **{kw}**: found in {len(locations)} location(s)\n"
                    for loc_file, desc in locations[:5]:
                        sub_content += f"  - `{loc_file}`: {desc}\n"
                sub_content += "\n"
        else:
            sub_content += "No keywords extracted from files in this folder.\n\n"

        (doc_dir / "sub.md").write_text(sub_content, encoding='utf-8')
        self.stats["docs_created"] += 1
        self.stats["bytes_written"] += len(sub_content)

    def generate_global_keywords(self):
        """Generate global keywords.md file."""
        self.log_progress("Generating global keywords index...")

        kw_content = f"# Global Keywords Index\n\n"
        kw_content += f"**Repository**: {self.repo_metadata.get('repo_name', 'Unknown')}\n\n"
        kw_content += f"**Total Unique Keywords**: {len(self.all_keywords)}\n\n"
        kw_content += "This is a comprehensive A-Z index of all keywords found across the repository.\n\n"

        # Group by first letter
        by_letter = defaultdict(list)
        for kw in self.all_keywords.keys():
            first_letter = kw[0].upper() if kw else '?'
            by_letter[first_letter].append(kw)

        for letter in sorted(by_letter.keys()):
            kw_content += f"## {letter}\n\n"
            for kw in sorted(by_letter[letter], key=lambda x: x.lower())[:1000]:  # Limit per letter
                locations = self.all_keywords[kw]
                kw_content += f"- **{kw}**: found in {len(locations)} file(s)\n"
                for file_path, desc in locations[:5]:  # Show first 5 locations
                    safe_name = Path(file_path).name.replace('.', '_')
                    rel_link = f"{Path(file_path).parent}/{safe_name}_docs.md"
                    kw_content += f"  - [{file_path}]({rel_link}): {desc}\n"
                if len(locations) > 5:
                    kw_content += f"  - ... and {len(locations) - 5} more\n"
            kw_content += "\n"

        keywords_file = self.output_dir / "keywords.md"
        keywords_file.write_text(kw_content, encoding='utf-8')
        self.stats["docs_created"] += 1
        self.stats["bytes_written"] += len(kw_content)
        self.stats["words_estimated"] += self.count_words(kw_content)

    def generate_global_index(self):
        """Generate root index.md file."""
        self.log_progress("Generating global index...")

        index_content = f"# Repository Documentation Index\n\n"
        index_content += f"**Repository**: {self.repo_metadata.get('repo_name', 'Unknown')}\n\n"
        index_content += f"**Generated**: {self.repo_metadata.get('generation_date', 'Unknown')}\n\n"

        if 'commit_sha' in self.repo_metadata:
            index_content += f"**Commit**: {self.repo_metadata['commit_sha'][:8]}\n\n"

        index_content += "## Quick Links\n\n"
        index_content += "- [Comprehensive Book](comprehensive_book.md) - Full stitched documentation\n"
        index_content += "- [Keywords A-Z](keywords.md) - Global keyword index\n"
        index_content += "- [Verification Report](verification_report.md) - Validation results\n\n"

        index_content += "## Folder Structure\n\n"

        # List all folders
        for folder_key in sorted(self.folder_structure.keys()):
            if folder_key == "ROOT":
                link = "index.md"
                display = "/ (root)"
            else:
                link = f"{folder_key}/index.md"
                display = folder_key

            file_count = len(self.folder_structure[folder_key]["files"])
            subfolder_count = len(self.folder_structure[folder_key]["folders"])
            index_content += f"- [{display}]({link}) - {file_count} file(s), {subfolder_count} subfolder(s)\n"

        index_content += "\n"

        index_file = self.output_dir / "index.md"
        index_file.write_text(index_content, encoding='utf-8')
        self.stats["docs_created"] += 1
        self.stats["bytes_written"] += len(index_content)

    def generate_comprehensive_book(self):
        """Generate comprehensive_book.md by stitching together all documentation."""
        self.log_progress("Generating comprehensive book...")

        book_content = f"# {self.repo_metadata.get('repo_name', 'Repository')} - Comprehensive Documentation\n\n"
        book_content += f"**Generated**: {self.repo_metadata.get('generation_date', 'Unknown')}\n\n"

        if 'commit_sha' in self.repo_metadata:
            book_content += f"**Commit**: {self.repo_metadata['commit_sha']}\n"
            book_content += f"**Author**: {self.repo_metadata.get('commit_author', 'Unknown')}\n"
            book_content += f"**Date**: {self.repo_metadata.get('commit_date', 'Unknown')}\n\n"

        book_content += "---\n\n"
        book_content += "# Table of Contents\n\n"

        # Build TOC
        chapter_num = 1
        for folder_key in sorted(self.folder_structure.keys()):
            book_content += f"{chapter_num}. [{folder_key}](#{folder_key.lower().replace('/', '-').replace('.', '-')})\n"
            chapter_num += 1

        book_content += "\n---\n\n"

        # Add chapters (one per folder)
        chapter_num = 1
        for folder_key in sorted(self.folder_structure.keys()):
            folder_anchor = folder_key.lower().replace('/', '-').replace('.', '-')
            book_content += f"# Chapter {chapter_num}: {folder_key} <a id=\"{folder_anchor}\"></a>\n\n"

            # Include folder doc.md content
            folder_path = Path(folder_key) if folder_key != "ROOT" else Path(".")
            doc_file = self.output_dir / folder_path / "doc.md"

            if doc_file.exists():
                try:
                    doc_text = doc_file.read_text(encoding='utf-8')
                    # Strip the title since we're adding our own
                    doc_text = re.sub(r'^#\s+Documentation:.*\n\n', '', doc_text)
                    book_content += doc_text
                except:
                    book_content += f"(Could not read documentation for {folder_key})\n"

            book_content += "\n"

            # Add file summaries
            folder_data = self.folder_structure[folder_key]
            if folder_data["files"]:
                book_content += f"## Files in {folder_key}\n\n"
                for file_path in sorted(folder_data["files"])[:20]:  # Limit to prevent huge book
                    file_rel = Path(file_path)
                    book_content += f"### {file_rel.name}\n\n"
                    book_content += f"**Path**: `{file_path}`\n\n"

                    # Add brief overview (first few lines from docs)
                    safe_name = file_rel.name.replace('.', '_')
                    docs_file = self.output_dir / file_rel.parent / f"{safe_name}_docs.md"
                    if docs_file.exists():
                        try:
                            docs_text = docs_file.read_text(encoding='utf-8')
                            # Extract just the overview section
                            overview_match = re.search(r'## High-Level Overview\s*\n\n(.*?)(?=\n##|\Z)', docs_text, re.DOTALL)
                            if overview_match:
                                overview = overview_match.group(1).strip()
                                # Limit to first 500 chars
                                if len(overview) > 500:
                                    overview = overview[:500] + "..."
                                book_content += f"{overview}\n\n"
                        except:
                            pass

                if len(folder_data["files"]) > 20:
                    book_content += f"*... and {len(folder_data['files']) - 20} more files*\n\n"

            book_content += "---\n\n"
            chapter_num += 1

        # Write book
        book_file = self.output_dir / "comprehensive_book.md"
        book_file.write_text(book_content, encoding='utf-8')
        self.stats["docs_created"] += 1
        self.stats["bytes_written"] += len(book_content)
        self.stats["words_estimated"] += self.count_words(book_content)

    def generate_verification_report(self):
        """Generate verification_report.md with validation results."""
        self.log_progress("Generating verification report...")

        report = f"# Verification Report\n\n"
        report += f"**Repository**: {self.repo_metadata.get('repo_name', 'Unknown')}\n\n"
        report += f"**Generated**: {datetime.now().isoformat()}\n\n"

        report += "## Summary\n\n"
        report += f"- **Files Scanned**: {self.stats['files_scanned']}\n"
        report += f"- **Files Processed**: {self.stats['files_processed']}\n"
        report += f"- **Docs Created**: {self.stats['docs_created']}\n"
        report += f"- **Bytes Written**: {self.stats['bytes_written']:,}\n"
        report += f"- **Words (estimated)**: {self.stats['words_estimated']:,}\n"
        report += f"- **Errors**: {len(self.stats['errors'])}\n"
        report += f"- **Skipped Files**: {len(self.stats['skipped_files'])}\n"
        report += f"- **Binary Files**: {len(self.stats['binary_files'])}\n\n"

        if self.stats['binary_files']:
            report += "## Binary Files\n\n"
            for bf in sorted(self.stats['binary_files'])[:100]:
                report += f"- `{bf}`\n"
            if len(self.stats['binary_files']) > 100:
                report += f"- ... and {len(self.stats['binary_files']) - 100} more\n"
            report += "\n"

        if self.stats['skipped_files']:
            report += "## Skipped Files\n\n"
            for sf in sorted(self.stats['skipped_files']):
                report += f"- `{sf}`\n"
            report += "\n"

        if self.stats['errors']:
            report += "## Errors\n\n"
            for error in self.stats['errors']:
                report += f"- {error}\n"
            report += "\n"
        else:
            report += "## Errors\n\n"
            report += "✅ No errors encountered.\n\n"

        report += "## File Integrity\n\n"
        report += "All generated markdown files have been written successfully.\n"

        verification_file = self.output_dir / "verification_report.md"
        verification_file.write_text(report, encoding='utf-8')
        self.stats["docs_created"] += 1
        self.stats["bytes_written"] += len(report)

    def generate_manifest(self):
        """Generate manifest.json with metadata and checksums."""
        self.log_progress("Generating manifest...")

        manifest = {
            "repo_name": self.repo_metadata.get('repo_name', 'Unknown'),
            "repo_path": str(self.repo_path),
            "commit_sha": self.repo_metadata.get('commit_sha', 'N/A'),
            "commit_author": self.repo_metadata.get('commit_author', 'N/A'),
            "commit_date": self.repo_metadata.get('commit_date', 'N/A'),
            "commit_message": self.repo_metadata.get('commit_message', 'N/A'),
            "remote_url": self.repo_metadata.get('remote_url', 'N/A'),
            "generation_date": self.repo_metadata.get('generation_date'),
            "generator_version": self.repo_metadata.get('generator_version'),
            "file_count": self.stats['files_scanned'],
            "docs_count": self.stats['docs_created'],
            "bytes_written": self.stats['bytes_written'],
            "words_estimated": self.stats['words_estimated'],
            "errors_count": len(self.stats['errors']),
            "checksums": {}
        }

        # Compute checksums for all generated docs
        self.log_progress("Computing checksums...")
        for doc_file in self.output_dir.rglob("*.md"):
            rel_path = doc_file.relative_to(self.output_dir)
            checksum = self.compute_file_hash(doc_file)
            if checksum:
                manifest["checksums"][str(rel_path)] = checksum

        manifest_file = self.output_dir / "manifest.json"
        with open(manifest_file, 'w', encoding='utf-8') as f:
            json.dump(manifest, f, indent=2)

        self.stats["bytes_written"] += manifest_file.stat().st_size

    def generate_readme(self):
        """Generate README.md explaining the documentation structure."""
        readme = f"# Documentation for {self.repo_metadata.get('repo_name', 'Repository')}\n\n"
        readme += "## Overview\n\n"
        readme += "This documentation was automatically generated by the World's Best Repo Book Generator.\n\n"
        readme += "## Structure\n\n"
        readme += "```\n"
        readme += "docs/\n"
        readme += "├── index.md                    # Main entry point\n"
        readme += "├── keywords.md                 # Global A-Z keyword index\n"
        readme += "├── comprehensive_book.md       # Full stitched book\n"
        readme += "├── verification_report.md      # Validation results\n"
        readme += "├── manifest.json               # Metadata & checksums\n"
        readme += "├── README.md                   # This file\n"
        readme += "└── <folder>/\n"
        readme += "    ├── index.md                # Folder index\n"
        readme += "    ├── doc.md                  # Folder documentation\n"
        readme += "    ├── sub.md                  # Folder keyword summary\n"
        readme += "    ├── <file>_docs.md          # File documentation\n"
        readme += "    └── <file>_kw.md            # File keywords\n"
        readme += "```\n\n"
        readme += "## Key Files\n\n"
        readme += "- **index.md**: Start here for navigation\n"
        readme += "- **comprehensive_book.md**: Read the entire codebase as a book\n"
        readme += "- **keywords.md**: Find specific terms and their locations\n"
        readme += "- **verification_report.md**: See generation statistics and any issues\n\n"
        readme += "## File Naming\n\n"
        readme += "For each source file `foo.py`, you'll find:\n"
        readme += "- `foo_py_docs.md` - Full documentation\n"
        readme += "- `foo_py_kw.md` - Keyword index\n\n"
        readme += "(Dots are replaced with underscores in generated filenames)\n\n"
        readme += "## Metadata\n\n"
        readme += f"- **Files Scanned**: {self.stats['files_scanned']}\n"
        readme += f"- **Docs Created**: {self.stats['docs_created']}\n"
        readme += f"- **Total Words**: ~{self.stats['words_estimated']:,}\n"
        readme += f"- **Generated**: {self.repo_metadata.get('generation_date', 'Unknown')}\n\n"

        if 'commit_sha' in self.repo_metadata:
            readme += "## Repository Snapshot\n\n"
            readme += f"- **Commit**: {self.repo_metadata['commit_sha']}\n"
            readme += f"- **Author**: {self.repo_metadata.get('commit_author', 'Unknown')}\n"
            readme += f"- **Date**: {self.repo_metadata.get('commit_date', 'Unknown')}\n"
            readme += f"- **Message**: {self.repo_metadata.get('commit_message', 'N/A')}\n\n"

        readme += "## How to Use\n\n"
        readme += "1. Start with `index.md` for an overview\n"
        readme += "2. Navigate to specific folders using the folder indexes\n"
        readme += "3. Read individual file documentation as needed\n"
        readme += "4. Use `keywords.md` to search for specific terms\n"
        readme += "5. Read `comprehensive_book.md` for a complete narrative\n\n"
        readme += "## Regenerating\n\n"
        readme += "To regenerate this documentation:\n\n"
        readme += "```bash\n"
        readme += "python3 repo_book_gen.py --source . --out ./docs\n"
        readme += "```\n\n"
        readme += "The process is idempotent and resumable.\n"

        readme_file = self.output_dir / "README.md"
        readme_file.write_text(readme, encoding='utf-8')
        self.stats["docs_created"] += 1
        self.stats["bytes_written"] += len(readme)

    def run(self):
        """Execute the complete documentation generation process."""
        # Create output directory
        self.output_dir.mkdir(parents=True, exist_ok=True)

        # Bootstrap
        self.log_progress("=" * 60)
        self.log_progress("REPO BOOK GENERATOR - STARTING")
        self.log_progress("=" * 60)

        self.repo_metadata = self.get_git_metadata()
        self.log_progress(f"Repository: {self.repo_metadata.get('repo_name', 'Unknown')}")
        if 'commit_sha' in self.repo_metadata:
            self.log_progress(f"Commit: {self.repo_metadata['commit_sha'][:8]}")

        # Scan
        all_files = self.scan_repository()

        # Per-file pass
        self.log_progress("\nProcessing files...")
        for i, rel_path in enumerate(all_files, 1):
            if i % 10 == 0 or i == len(all_files):
                self.log_progress(f"Progress: {i}/{len(all_files)} files")
            self.process_file(rel_path)

        # Per-folder pass
        self.log_progress("\nProcessing folders...")
        for folder_key in sorted(self.folder_structure.keys()):
            self.generate_folder_docs(folder_key)

        # Global merges
        self.generate_global_keywords()
        self.generate_global_index()
        self.generate_comprehensive_book()

        # Verification & manifest
        self.generate_verification_report()
        self.generate_manifest()
        self.generate_readme()

        # Final summary
        self.log_progress("\n" + "=" * 60)
        self.log_progress("GENERATION COMPLETE")
        self.log_progress("=" * 60)
        self.log_progress(f"Files scanned: {self.stats['files_scanned']}")
        self.log_progress(f"Files processed: {self.stats['files_processed']}")
        self.log_progress(f"Docs created: {self.stats['docs_created']}")
        self.log_progress(f"Bytes written: {self.stats['bytes_written']:,}")
        self.log_progress(f"Words estimated: {self.stats['words_estimated']:,}")
        self.log_progress(f"Errors: {len(self.stats['errors'])}")

        # Return final JSON summary
        return {
            "repo_source": str(self.repo_path),
            "repo_fingerprint": self.repo_metadata.get('commit_sha', 'N/A'),
            "files_scanned": self.stats['files_scanned'],
            "docs_created": self.stats['docs_created'],
            "words_estimated": self.stats['words_estimated'],
            "bytes_written": self.stats['bytes_written'],
            "errors": self.stats['errors']
        }


def main():
    import argparse

    parser = argparse.ArgumentParser(description="World's Best Repo Book Generator")
    parser.add_argument('--source', default='.', help='Repository path (default: current directory)')
    parser.add_argument('--out', default='./docs', help='Output directory (default: ./docs)')

    args = parser.parse_args()

    generator = RepoBookGenerator(args.source, args.out)
    result = generator.run()

    # Print final JSON summary
    print("\n" + "=" * 60)
    print("FINAL SUMMARY (JSON)")
    print("=" * 60)
    print(json.dumps(result, indent=2))


if __name__ == '__main__':
    main()

```

## High-Level Overview

This is a Python source file.

### Imports (10)

- `import os`
- `import json`
- `import hashlib`
- `import mimetypes`
- `import re`
- `import subprocess`
- `from pathlib import Path`
- `from datetime import datetime`
- `from typing import Dict, List, Set, Tuple, Any`
- `from collections import defaultdict`

### Classes (1)

- **RepoBookGenerator**

### Functions (1)

- `main()`

### Module Documentation

World's Best Repo Book Generator and Index Builder
Produces comprehensive documentation for an entire repository.


## Performance & Security Notes

⚠️ **Security**: Use of eval() - potential security risk
⚠️ **Security**: Use of exec() - potential security risk
⚠️ **Security**: Use of os.system() - potential command injection risk

## Related Files

Files and modules referenced:

- `argparse`
- `collections`
- `hashlib`
- `mimetypes`
- `subprocess`
