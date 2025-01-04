# PYTHON
from flask import Flask, render_template, request, jsonify, send_from_directory
import os
import pandas as pd
import json
from datetime import datetime

app = Flask(__name__)

# Folder Configurations
UPLOAD_FOLDER = 'uploads'
PROCESSED_FOLDER = 'processed_files'
ALLOWED_EXTENSIONS = {'xls', 'xlsx'}
os.makedirs(UPLOAD_FOLDER, exist_ok=True)
os.makedirs(PROCESSED_FOLDER, exist_ok=True)
app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER

# Helper function to check allowed file extensions
def allowed_file(filename):
    return '.' in filename and filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS

# Home route
@app.route('/')
def index():
    return render_template('Export_files mangemant.html')

# Upload route
@app.route('/upload', methods=['POST'])
def upload_file():
    if 'fileUpload' not in request.files:
        return jsonify({'error': 'No file part'})

    file = request.files['fileUpload']
    if file.filename == '':
        return jsonify({'error': 'No selected file'})

    if allowed_file(file.filename):
        # Save file with timestamp
        timestamp = datetime.now().strftime('%Y%m%d%H%M%S')
        file_path = os.path.join(app.config['UPLOAD_FOLDER'], f"{timestamp}_{file.filename}")
        file.save(file_path)
        
        # Process the uploaded Excel file
        process_excel_file(file_path)
        return jsonify({'message': f'File uploaded and processed successfully: {file.filename}', 
                        'file_path': file_path})
    else:
        return jsonify({'error': 'Invalid file format. Only .xls or .xlsx files are allowed.'})

# Load majors and fields from JSON
@app.route('/load_json', methods=['POST'])
def load_json():
    data = request.json
    try:
        # Save incoming JSON data to majors.json
        with open(os.path.join(PROCESSED_FOLDER, 'majors.json'), 'w') as f:
            json.dump(data, f, indent=2)
        return jsonify({'message': 'JSON file saved successfully'})
    except Exception as e:
        return jsonify({'error': f'Failed to save JSON: {e}'})

# Process Excel files dynamically based on JSON fields
def process_excel_file(file_path):
    try:
        # Load majors.json for filtering
        json_file = os.path.join(PROCESSED_FOLDER, 'majors.json')
        if not os.path.exists(json_file):
            print('majors.json not found. Skipping filtering.')
            return

        with open(json_file, 'r') as f:
            data = json.load(f)

        # Extract majors and fields from JSON
        majors = [item['major'] for item in data]
        fields = sum([item['fields'] for item in data], [])
        
        # Read Excel file
        df = pd.read_excel(file_path, engine='openpyxl')
        
        # Match columns based on JSON fields
        matching_columns = [col for col in fields if col in df.columns]
        
        if matching_columns:
            # Filter data and drop duplicates
            filtered_data = df[matching_columns].drop_duplicates()
            
            # Create output format
            output = []
            for item in data:
                filtered_fields = [col for col in item['fields'] if col in matching_columns]
                if filtered_fields:
                    output.append({
                        "major": item['major'],
                        "fields": filtered_fields
                    })
            
            # Print JSON output
            print(json.dumps(output, indent=2))
            
            # Save filtered Excel file
            file_name = os.path.splitext(os.path.basename(file_path))[0]
            output_file = os.path.join(PROCESSED_FOLDER, f"{file_name}_filtered.xlsx")
            filtered_data.to_excel(output_file, index=False)
            print(f'Processed and saved: {output_file}')
            
            # Return JSON response
            return jsonify(output)
        else:
            print('No matching columns found.')
            return jsonify({"message": "No matching columns found to process."})

    except Exception as e:
        return jsonify({"error": f"Error processing file: {e}"})

# Download processed files
@app.route('/download/<filename>')
def download_file(filename):
    return send_from_directory(PROCESSED_FOLDER, filename, as_attachment=True)

if __name__ == '__main__':
    app.run(debug=True)
