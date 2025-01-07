# studentapp
programm for traking the productiviti
DATABASE_URI
sqlite:///sams.db
from flask import Flask, request, jsonify, send_file
from flask_sqlalchemy import SQLAlchemy
from marshmallow import Schema, fields, ValidationError
from dotenv import load_dotenv
import matplotlib.pyplot as plt
from io import BytesIO
import csv
import os
import logging
from datetime import datetime

# Load environment variables
load_dotenv()

# Initialize Flask App
app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = os.getenv('DATABASE_URI', 'sqlite:///sams.db')
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

# Initialize Database
db = SQLAlchemy(app)

# Configure Logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

# Models
class Student(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    city = db.Column(db.String(100))
    country = db.Column(db.String(100))

class Application(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    student_id = db.Column(db.Integer, db.ForeignKey('student.id'), nullable=False)
    course_name = db.Column(db.String(100), nullable=False)
    intake_date = db.Column(db.Date, nullable=False)
    interview_passed = db.Column(db.Boolean, default=False)
    written_test_passed = db.Column(db.Boolean, default=False)
    document_submitted = db.Column(db.DateTime)
    unconditional_offer = db.Column(db.Boolean, default=False)

    student = db.relationship('Student', backref=db.backref('applications', lazy=True))

# Schemas for Input Validation
class StudentSchema(Schema):
    name = fields.String(required=True)
    city = fields.String(required=True)
    country = fields.String(required=True)

class ApplicationSchema(Schema):
    student_id = fields.Integer(required=True)
    course_name = fields.String(required=True)
    intake_date = fields.Date(required=True)

class UpdateApplicationSchema(Schema):
    interview_passed = fields.Boolean()
    written_test_passed = fields.Boolean()
    document_submitted = fields.DateTime()
    unconditional_offer = fields.Boolean()

# Routes
@app.route('/add_student', methods=['POST'])
def add_student():
    try:
        data = StudentSchema().load(request.json)
        student = Student(**data)
        db.session.add(student)
        db.session.commit()
        logging.info(f"Student added: {student.name}")
        return jsonify({'message': 'Student added successfully', 'student_id': student.id}), 201
    except ValidationError as e:
        logging.error(f"Validation Error: {e.messages}")
        return jsonify({'error': e.messages}), 400
    except Exception as e:
        logging.error(f"Error adding student: {str(e)}")
        return jsonify({'error': 'Failed to add student'}), 500

@app.route('/add_application', methods=['POST'])
def add_application():
    try:
        data = ApplicationSchema().load(request.json)
        application = Application(**data)
        db.session.add(application)
        db.session.commit()
        logging.info(f"Application added for student ID: {data['student_id']}")
        return jsonify({'message': 'Application added successfully', 'application_id': application.id}), 201
    except ValidationError as e:
        logging.error(f"Validation Error: {e.messages}")
        return jsonify({'error': e.messages}), 400
    except Exception as e:
        logging.error(f"Error adding application: {str(e)}")
        return jsonify({'error': 'Failed to add application'}), 500

@app.route('/update_application/<int:application_id>', methods=['PUT'])
def update_application(application_id):
    try:
        data = UpdateApplicationSchema().load(request.json)
        application = Application.query.get_or_404(application_id)

        for key, value in data.items():
            setattr(application, key, value)

        db.session.commit()
        logging.info(f"Application updated: ID {application_id}")
        return jsonify({'message': 'Application updated successfully'})
    except ValidationError as e:
        logging.error(f"Validation Error: {e.messages}")
        return jsonify({'error': e.messages}), 400
    except Exception as e:
        logging.error(f"Error updating application: {str(e)}")
        return jsonify({'error': 'Failed to update application'}), 500

@app.route('/generate_report', methods=['GET'])
def generate_report():
    try:
        applications = Application.query.all()
        courses = [app.course_name for app in applications]
        course_counts = {course: courses.count(course) for course in set(courses)}

        plt.bar(course_counts.keys(), course_counts.values())
        plt.xlabel('Courses')
        plt.ylabel('Number of Applications')
        plt.title('Applications per Course')

        img = BytesIO()
        plt.savefig(img, format='png')
        img.seek(0)
        plt.close()
        logging.info("Report generated")
        return send_file(img, mimetype='image/png')
    except Exception as e:
        logging.error(f"Error generating report: {str(e)}")
        return jsonify({'error': 'Failed to generate report'}), 500

@app.route('/export_data', methods=['GET'])
def export_data():
    try:
        students = Student.query.all()
        applications = Application.query.all()

        csv_file = BytesIO()
        writer = csv.writer(csv_file)
        writer.writerow(['Type', 'ID', 'Name/City/Country/Student_ID', 'Course/Intake/Interview', 'Written_Test/Document/Offer'])

        for student in students:
            writer.writerow(['Student', student.id, student.name, student.city, student.country])

        for app in applications:
            writer.writerow(['Application', app.id, app.student_id, app.course_name, app.intake_date,
                             app.interview_passed, app.written_test_passed, app.document_submitted, app.unconditional_offer])

        csv_file.seek(0)
        logging.info("Data exported")
        return send_file(csv_file, mimetype='text/csv', as_attachment=True, download_name='exported_data.csv')
    except Exception as e:
        logging.error(f"Error exporting data: {str(e)}")
        return jsonify({'error': 'Failed to export data'}), 500

# Database Initialization
@app.before_first_request
def create_tables():
    db.create_all()

# Run App
if __name__ == '__main__':
    app.run(debug=True)

Flask
Flask-SQLAlchemy
marshmallow
marshmallow-sqlalchemy
python-dotenv
matplotlib

pip install -r requirements.txt

pip install gunicorn
gunicorn app:app

DATABASE_URI=sqlite:///sams.db  # Update this for PostgreSQL/MySQL in production
FLASK_ENV=production
SECRET_KEY=your_secret_key

web: gunicorn app:app

FROM python:3.9-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["gunicorn", "app:app", "--bind", "0.0.0.0:5000"]

docker build -t flask-app .
docker run -p 5000:5000 flask-app
