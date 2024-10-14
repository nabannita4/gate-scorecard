# gate-scorecard
import os
import re
import pandas as pd
import fitz  # PyMuPDF
from flask import Flask, render_template, request, jsonify
from werkzeug.utils import secure_filename

UPLOAD_FOLDER = 'uploaded_files'
PROCESSED_FOLDER = 'processed_files'

app = Flask(__name__)
app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER
app.config['PROCESSED_FOLDER'] = PROCESSED_FOLDER

def extract_text_from_pdf(file_path):
    """Extract text from a PDF file using PyMuPDF."""
    text = ""
    try:
        with fitz.open(file_path) as doc:
            for page in doc:
                text += page.get_text()
    except Exception as e:
        print(f"Could not extract text from {file_path}: {e}")
    return text

def determine_exam_type(text):
    """Determine the exam type based on the extracted text."""
    if re.search(r'Data\s*Science\s*and\s*Artificial\s*Intelligence\s*\(DA\)', text, re.IGNORECASE):
        return "GATE_DA"
    elif re.search(r'COMPUTER\s*SCIENCE\s*AND\s*INFORMATION\s*TECHNOLOGY\s*\(CS\)', text, re.IGNORECASE):
        return "GATE_CS"
    return None

def parse_gate_data(text):
    """Parse GATE data using string matching."""
    data = {}

    try:
        # Remove newlines to make matching more consistent
        text = text.replace('\n', ' ')

        # Extract registration number
        reg_no_match = re.search(r'Registration\s*No.\s*:\s*([A-Z0-9]+)', text, re.IGNORECASE)
        if reg_no_match:
            data["Registration No."] = reg_no_match.group(1)

        # Extract candidate name
        name_match = re.search(r'Name\s*of\s*the\s*Candidate\s*:\s*([A-Za-z ]+)', text, re.IGNORECASE)
        if name_match:
            data["Name of the Candidate"] = name_match.group(1).strip()

        # Extract marks
        marks_match = re.search(r'Marks\s*out\s*of\s*100\s*:?\s*([\d.]+)', text, re.IGNORECASE)
        if marks_match:
            data["Marks out of 100"] = marks_match.group(1)

        # Extract GATE score
        score_match = re.search(r'GATE\s*Score\s*:?\s*([\d.]+)', text, re.IGNORECASE)
        if score_match:
            data["GATE Score"] = score_match.group(1)

        # Extract All India Rank (AIR)
        rank_match = re.search(r'All\s*India\s*Rank\s*\(AIR\)\s*:?\s*(\d+)', text, re.IGNORECASE)
        if rank_match:
            data["All India Rank (AIR)"] = rank_match.group(1)

    except Exception as e:
        print(f"Error parsing GATE data: {e}")

    return data

def save_data_to_excel(data, filename):
    """Save the parsed data to an Excel file."""
    if data:
        df = pd.DataFrame(data)
        columns = ["Registration No.", "Name of the Candidate", "Marks out of 100", "GATE Score", "All India Rank (AIR)"]
        df = df.reindex(columns=columns)

        print(f"Attempting to save data to {filename}")
        print("Data to be saved:\n", df)

        try:
            df.to_excel(filename, index=False)
            print(f"Data saved successfully to {filename}")
        except Exception as e:
            print(f"Error saving Excel file to {filename}: {e}")
    else:
        print(f"No data to save for {filename}")

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/upload', methods=['POST'])
def upload_files():
    if 'files[]' not in request.files:
        return jsonify({'status': 'error', 'message': 'No files part in the request'})

    files = request.files.getlist('files[]')
    if not files:
        return jsonify({'status': 'error', 'message': 'No files selected'})

    gate_cs_data = []
    gate_da_data = []

    # Ensure directories exist
    os.makedirs(app.config['UPLOAD_FOLDER'], exist_ok=True)
    os.makedirs(app.config['PROCESSED_FOLDER'], exist_ok=True)

    for file in files:
        if file and file.filename.endswith('.pdf'):
            filename = secure_filename(file.filename)
            file_path = os.path.join(app.config['UPLOAD_FOLDER'], filename)

            try:
                file.save(file_path)
                print(f"File saved to {file_path}")
            except Exception as e:
                return jsonify({'status': 'error', 'message': f'Failed to save file: {str(e)}'})

            text = extract_text_from_pdf(file_path)
            if text:
                print("Extracted text:\n", text)

                parsed_data = parse_gate_data(text)
                print("Parsed data:", parsed_data)

                if parsed_data:
                    reg_no = parsed_data.get("Registration No.")
                    exam_type = determine_exam_type(text)

                    # If registration number starts with 'CS', treat it as GATE CS
                    if reg_no and reg_no.upper().startswith('CS'):
                        exam_type = "GATE_CS"

                    print("Final exam type:", exam_type)

                    if exam_type == "GATE_CS":
                        gate_cs_data.append(parsed_data)
                    elif exam_type == "GATE_DA":
                        gate_da_data.append(parsed_data)

    try:
        if gate_cs_data:
            excel_path = os.path.join(app.config['PROCESSED_FOLDER'], 'GATE_CS_Scorecards.xlsx')
            save_data_to_excel(gate_cs_data, excel_path)

        if gate_da_data:
            excel_path = os.path.join(app.config['PROCESSED_FOLDER'], 'GATE_DA_Scorecards.xlsx')
            save_data_to_excel(gate_da_data, excel_path)

    except Exception as e:
        print(f"Error saving Excel files: {str(e)}")
        return jsonify({'status': 'error', 'message': f"Error saving Excel files: {str(e)}"})

    return jsonify({'status': 'success', 'message': 'Files uploaded and processed successfully!'})

if __name__ == '__main__':
    os.makedirs(UPLOAD_FOLDER, exist_ok=True)
    os.makedirs(PROCESSED_FOLDER, exist_ok=True)
    app.run(debug=True)
