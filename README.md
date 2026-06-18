import pandas as pd
import gradio as gr

# --- 1. Data Structures: Simulate Student, Lecturer, and Grade Data ---

# Students DataFrame: ID, Email, Name, College
students_df = pd.DataFrame({
    'student_id': ['S001', 'S002', 'S003'],
    'email': ['student1@college.edu', 'student2@college.edu', 'student3@college.edu'],
    'password': ['pass1', 'pass2', 'pass3'], # For simulation, in a real app, this would be hashed
    'name': ['Alice Smith', 'Bob Johnson', 'Charlie Brown'],
    'college': ['Engineering', 'Arts & Science', 'Business']
})

# Lecturers DataFrame: ID, Email, Name, College
lecturers_df = pd.DataFrame({
    'lecturer_id': ['L001', 'L002'],
    'email': ['lecturer1@college.edu', 'lecturer2@college.edu'],
    'password': ['lectpass1', 'lectpass2'], # For simulation
    'name': ['Dr. Emily White', 'Prof. David Green'],
    'college': ['Engineering', 'Arts & Science']
})

# Grades DataFrame: student_id, course, grade, lecturer_id
grades_df = pd.DataFrame(columns=['student_id', 'course', 'grade', 'lecturer_id'])

# --- Initial Data for Demonstration (if not already loaded) ---
# This assumes the demo data from earlier cells is to be included for a runnable app
# If you want a fresh start, comment these lines out
# Example of adding demo grades if grades_df is empty
if grades_df.empty:
    new_grades = [
        {'student_id': 'S001', 'course': 'Calculus I', 'grade': 90, 'lecturer_id': 'L001'},
        {'student_id': 'S002', 'course': 'Physics', 'grade': 85, 'lecturer_id': 'L001'},
        {'student_id': 'S001', 'course': 'Linear Algebra', 'grade': 78, 'lecturer_id': 'L001'},
        {'student_id': 'S003', 'course': 'Chemistry', 'grade': 92, 'lecturer_id': 'L001'}
    ]
    grades_df = pd.concat([grades_df, pd.DataFrame(new_grades)], ignore_index=True)

# --- 2. User Authentication Functions ---

def student_login(student_id, email, password):
    user = students_df[(students_df['student_id'] == student_id) & (students_df['email'] == email) & (students_df['password'] == password)]
    if not user.empty:
        return user.iloc[0]
    else:
        return None

def lecturer_login(lecturer_id, email, password):
    user = lecturers_df[(lecturers_df['lecturer_id'] == lecturer_id) & (lecturers_df['email'] == email) & (lecturers_df['password'] == password)]
    if not user.empty:
        return user.iloc[0]
    else:
        return None

# --- 3. Lecturer Functions: Grade Input (Original, not Gradio wrapper) ---
def input_grade(lecturer_info, student_id, course, grade):
    global grades_df
    if lecturer_info is None:
        print("Error: Lecturer not logged in.")
        return False

    if student_id not in students_df['student_id'].values:
        print(f"Error: Student ID '{student_id}' not found.")
        return False

    new_grade = pd.DataFrame([{'student_id': student_id, 'course': course, 'grade': grade, 'lecturer_id': lecturer_info['lecturer_id']}]
    )
    grades_df = pd.concat([grades_df, new_grade], ignore_index=True)
    return True

# --- 4. Student Functions: Grade Checking and CGPA Calculation (Original, not Gradio wrapper) ---
def check_grades(student_info):
    if student_info is None:
        return None

    student_grades = grades_df[grades_df['student_id'] == student_info['student_id']]
    return student_grades

def calculate_cgpa(student_grades):
    if student_grades is None or student_grades.empty:
        return 0.0

    # Simple average for demonstration; real CGPA calculation is more complex with credit hours
    average_grade = student_grades['grade'].mean()

    # Convert percentage grade to a 4.0 scale CGPA (very simplified)
    if 90 <= average_grade <= 100:
        cgpa = 4.0
    elif 80 <= average_grade < 90:
        cgpa = 3.0
    elif 70 <= average_grade < 80:
        cgpa = 2.0
    elif 60 <= average_grade < 70:
        cgpa = 1.0
    else:
        cgpa = 0.0

    return cgpa

# --- 5. College Address (Placeholder) ---
college_addresses = {
    'Engineering': '123 Engineering Way, University City, State 12345',
    'Arts & Science': '456 Liberal Arts Blvd, University City, State 12345',
    'Business': '789 Business School Dr, University City, State 12345'
}

def get_college_address(college_name):
    return college_addresses.get(college_name, 'Address not found for this college.')

# --- Gradio Wrapper Functions ---

def get_student_add_controls_update(interactive_state):
    return (
        gr.update(interactive=interactive_state), # student_id_add
        gr.update(interactive=interactive_state), # student_email_add
        gr.update(interactive=interactive_state), # student_password_add
        gr.update(interactive=interactive_state), # student_name_add
        gr.update(interactive=interactive_state), # student_college_add
        gr.update(interactive=interactive_state)  # add_student_btn
    )

def get_student_modify_controls_update(interactive_state):
    return (
        gr.update(interactive=interactive_state), # student_id_select_modify
        gr.update(interactive=interactive_state), # student_email_modify
        gr.update(interactive=interactive_state), # student_password_modify
        gr.update(interactive=interactive_state), # student_name_modify
        gr.update(interactive=interactive_state), # student_college_modify
        gr.update(interactive=interactive_state)  # modify_student_btn
    )

def gradio_student_login(student_id_input, email_input, password_input, logged_in_student_session):
    user = students_df[(
        students_df['student_id'] == student_id_input) &
                       (students_df['email'] == email_input) &
                       (students_df['password'] == password_input)]
    if not user.empty:
        return f"Student {user.iloc[0]['name']} logged in successfully.", user.iloc[0]
    else:
        return "Invalid student ID, email, or password.", pd.Series(dtype='object') # Return empty Series for invalid login

def gradio_lecturer_login(lecturer_id_input, email_input, password_input):
    user = lecturers_df[(
        lecturers_df['lecturer_id'] == lecturer_id_input) &
                        (lecturers_df['email'] == email_input) &
                        (lecturers_df['password'] == password_input)]
    if not user.empty:
        return (f"Lecturer {user.iloc[0]['name']} logged in successfully.",
                user.iloc[0], # The logged in lecturer object for the gr.State
                gr.update(choices=students_df['student_id'].tolist())) # Update choices for dropdown
    else:
        return ("Invalid lecturer ID, email, or password.",
                pd.Series(dtype='object'), # Empty Series for invalid login
                gr.update(choices=[])) # Clear choices for dropdown

def gradio_input_grade(logged_in_lecturer_session, student_id_input, course_input, grade_input):
    global grades_df
    if logged_in_lecturer_session.empty: # Check if the Series is empty
        return "Error: Please log in as a lecturer first.", grades_df # Return current grades_df

    try:
        grade_input = int(grade_input)
    except ValueError:
        return "Error: Grade must be a number.", grades_df

    if student_id_input not in students_df['student_id'].values:
        return f"Error: Student ID '{student_id_input}' not found.", grades_df

    new_grade_entry = pd.DataFrame([{
        'student_id': student_id_input,
        'course': course_input,
        'grade': grade_input,
        'lecturer_id': logged_in_lecturer_session['lecturer_id']
    }])
    grades_df = pd.concat([grades_df, new_grade_entry], ignore_index=True)
    return f"Grade '{grade_input}' for course '{course_input}' added for student '{student_id_input}'.", grades_df

def gradio_check_grades(logged_in_student_session):
    if logged_in_student_session.empty: # Check if the Series is empty
        return "Error: Please log in as a student first.", pd.DataFrame(columns=['course', 'grade'])

    student_grades = grades_df[grades_df['student_id'] == logged_in_student_session['student_id']]
    if student_grades.empty:
        return f"No grades found for {logged_in_student_session['name']}.", pd.DataFrame(columns=['course', 'grade'])
    else:
        cgpa = calculate_cgpa(student_grades)
        message = f"Grades for {logged_in_student_session['name']} ({logged_in_student_session['student_id']})\nAverage CGPA: {cgpa:.2f}"
        return message, student_grades[['course', 'grade']]

def gradio_get_college_address(college_name_input):
    return get_college_address(college_name_input)

# New function for adding student
def gradio_add_student(logged_in_lecturer_session, student_id, email, password, name, college):
    global students_df
    if logged_in_lecturer_session.empty: # Check if the Series is empty
        return "Error: Lecturer not logged in to add student.", gr.update(value=students_df), gr.update(choices=students_df['student_id'].tolist())

    # Check if student ID or email already exists
    if student_id in students_df['student_id'].values:
        return f"Error: Student with ID '{student_id}' already exists.", gr.update(value=students_df), gr.update(choices=students_df['student_id'].tolist())
    if email in students_df['email'].values:
        return f"Error: Student with email '{email}' already exists.", gr.update(value=students_df), gr.update(choices=students_df['student_id'].tolist())

    new_student = pd.DataFrame([{
        'student_id': student_id,
        'email': email,
        'password': password, # In a real app, hash this!
        'name': name,
        'college': college
    }])
    students_df = pd.concat([students_df, new_student], ignore_index=True)
    return (f"Student '{name}' (ID: {student_id}) added successfully.",
            gr.update(value=students_df),
            gr.update(choices=students_df['student_id'].tolist())) # Return updated students_df and update dropdown choices

def gradio_modify_student_populate_fields(selected_student_id):
    if selected_student_id and selected_student_id in students_df['student_id'].values:
        student_data = students_df[students_df['student_id'] == selected_student_id].iloc[0]
        return (
            gr.update(value=student_data['email']),
            gr.update(value=student_data['password']),
            gr.update(value=student_data['name']),
            gr.update(value=student_data['college'])
        )
    return (
        gr.update(value=""),
        gr.update(value=""),
        gr.update(value=""),
        gr.update(value="")
    )

def gradio_modify_student(logged_in_lecturer_session, student_id_to_modify, new_email, new_password, new_name, new_college):
    global students_df
    if logged_in_lecturer_session.empty:
        return "Error: Lecturer not logged in to modify student.", gr.update(value=students_df), gr.update(choices=students_df['student_id'].tolist())

    if student_id_to_modify not in students_df['student_id'].values:
        return f"Error: Student with ID '{student_id_to_modify}' not found.", gr.update(value=students_df), gr.update(choices=students_df['student_id'].tolist())

    # Check for email uniqueness if email is changed to a different existing email
    if new_email != students_df[students_df['student_id'] == student_id_to_modify]['email'].iloc[0] and new_email in students_df['email'].values:
        return f"Error: Email '{new_email}' already belongs to another student.", gr.update(value=students_df), gr.update(choices=students_df['student_id'].tolist())

    students_df.loc[students_df['student_id'] == student_id_to_modify, 'email'] = new_email
    students_df.loc[students_df['student_id'] == student_id_to_modify, 'password'] = new_password # Hash in real app
    students_df.loc[students_df['student_id'] == student_id_to_modify, 'name'] = new_name
    students_df.loc[students_df['student_id'] == student_id_to_modify, 'college'] = new_college

    return (f"Student '{new_name}' (ID: {student_id_to_modify}) updated successfully.",
            gr.update(value=students_df),
            gr.update(choices=students_df['student_id'].tolist()))

# --- Gradio Interface Layout ---

with gr.Blocks(title="Student Grade Management App") as demo:
    gr.Markdown("# Student Grade Management System")
    gr.Image(value="/content/72d26c7c-f7cc-46c9-a1bd-08a1215591ce.jpg", label="Conceptual Logo", width=200, height=200)
    gr.Markdown("""This is a conceptual prototype.""")

    # Use gr.State to maintain logged-in user info across interactions
    logged_in_student_state = gr.State(value=pd.Series(dtype='object')) # Initialize with empty Series
    logged_in_lecturer_state = gr.State(value=pd.Series(dtype='object')) # Initialize with empty Series

    with gr.Tab("Student Portal"):
        gr.Markdown("### Student Login")
        with gr.Row():
            student_id_input = gr.Textbox(label="Student ID", placeholder="e.g., S001")
            student_email_input = gr.Textbox(label="Email", placeholder="e.g., student1@college.edu")
            student_password_input = gr.Textbox(label="Password", type="password", placeholder="e.g., pass1")
        student_login_btn = gr.Button("Student Login")
        student_login_output = gr.Textbox(label="Login Status", interactive=False)

        student_login_btn.click(
            gradio_student_login,
            inputs=[student_id_input, student_email_input, student_password_input, logged_in_student_state],
            outputs=[student_login_output, logged_in_student_state]
        )

        gr.Markdown("### Check My Grades")
        check_grades_btn = gr.Button("Show My Grades and CGPA")
        student_grades_output = gr.Textbox(label="Your Grades & CGPA", interactive=False)
        student_grades_dataframe = gr.Dataframe(headers=["Course", "Grade"], label="Detailed Grades", interactive=False)

        check_grades_btn.click(
            gradio_check_grades,
            inputs=[logged_in_student_state],
            outputs=[student_grades_output, student_grades_dataframe]
        )

    with gr.Tab("Lecturer Portal"):
        gr.Markdown("### Lecturer Login")
        with gr.Row():
            lecturer_id_input = gr.Textbox(label="Lecturer ID", placeholder="e.g., L001")
            lecturer_email_input = gr.Textbox(label="Email", placeholder="e.g., lecturer1@college.edu")
            lecturer_password_input = gr.Textbox(label="Password", type="password", placeholder="e.g., lectpass1")
        lecturer_login_btn = gr.Button("Lecturer Login")
        lecturer_login_output = gr.Textbox(label="Login Status", interactive=False)

        # Display all students at the top of the Lecturer Portal
        gr.Markdown("### All Students in System")
        all_students_display = gr.Dataframe(value=students_df, label="All Students in System", interactive=False, row_count=(1, 'dynamic'), col_count=(5, 'fixed'))

        gr.Markdown("### Add New Student")
        with gr.Row():
            student_id_add = gr.Textbox(label="New Student ID", placeholder="e.g., S004", interactive=True)
            student_email_add = gr.Textbox(label="New Student Email", placeholder="e.g., student4@college.edu", interactive=True)
        with gr.Row():
            student_password_add = gr.Textbox(label="New Student Password", type="password", placeholder="e.g., pass4", interactive=True)
            student_name_add = gr.Textbox(label="New Student Name", placeholder="e.g., David Lee", interactive=True)
            student_college_add = gr.Dropdown(label="College", choices=['Engineering', 'Arts & Science', 'Business'], interactive=True)
        add_student_btn = gr.Button("Add Student to System", interactive=True)
        add_student_output = gr.Textbox(label="Student Addition Status", interactive=False)

        gr.Markdown("### Modify Student Information")
        student_id_select_modify = gr.Dropdown(label="Select Student ID to Modify", choices=students_df['student_id'].tolist(), interactive=True)
        with gr.Row():
            student_email_modify = gr.Textbox(label="New Student Email", placeholder="e.g., student_new@college.edu", interactive=True)
            student_password_modify = gr.Textbox(label="New Student Password", type="password", placeholder="new_pass", interactive=True)
        with gr.Row():
            student_name_modify = gr.Textbox(label="New Student Name", placeholder="e.g., Jane Doe", interactive=True)
            student_college_modify = gr.Dropdown(label="New College", choices=['Engineering', 'Arts & Science', 'Business'], interactive=True)
        modify_student_btn = gr.Button("Modify Student Information", interactive=True)
        modify_student_output = gr.Textbox(label="Student Modification Status", interactive=False)

        lecturer_login_btn.click(
            gradio_lecturer_login,
            inputs=[lecturer_id_input, lecturer_email_input, lecturer_password_input],
            outputs=[
                lecturer_login_output,
                logged_in_lecturer_state,
                student_id_select_modify # Only updating choices here now
            ]
        )

        # Populate modification fields when a student is selected
        student_id_select_modify.change(
            gradio_modify_student_populate_fields,
            inputs=[student_id_select_modify],
            outputs=[student_email_modify, student_password_modify, student_name_modify, student_college_modify]
        )

        add_student_btn.click(
            gradio_add_student,
            inputs=[
                logged_in_lecturer_state, student_id_add, student_email_add,
                student_password_add, student_name_add, student_college_add
            ],
            outputs=[add_student_output, all_students_display, student_id_select_modify] # Update choices for dropdown
        )

        modify_student_btn.click(
            gradio_modify_student,
            inputs=[
                logged_in_lecturer_state, student_id_select_modify, student_email_modify,
                student_password_modify, student_name_modify, student_college_modify
            ],
            outputs=[modify_student_output, all_students_display, student_id_select_modify] # Update choices for dropdown
        )

        gr.Markdown("### Input Student Grades")
        with gr.Row():
            grade_student_id = gr.Textbox(label="Student ID", placeholder="e.g., S001")
            grade_course = gr.Textbox(label="Course Name", placeholder="e.g., Calculus I")
            grade_value = gr.Number(label="Grade (0-100)", minimum=0, maximum=100, step=1)
        input_grade_btn = gr.Button("Add Grade")
        input_grade_output = gr.Textbox(label="Grade Input Status", interactive=False)
        all_grades_display = gr.Dataframe(value=grades_df, label="All Grades in System", interactive=False, row_count=(1, 'dynamic'), col_count=(4, 'fixed'))

        input_grade_btn.click(
            gradio_input_grade,
            inputs=[logged_in_lecturer_state, grade_student_id, grade_course, grade_value],
            outputs=[input_grade_output, all_grades_display]
        )

    with gr.Tab("College Information"):
        gr.Markdown("### College Address Lookup")
        college_name_input = gr.Textbox(label="College Name", placeholder="e.g., Engineering")
        get_address_btn = gr.Button("Get Address")
        college_address_output = gr.Textbox(label="College Address", interactive=False)

        get_address_btn.click(
            gradio_get_college_address,
            inputs=[college_name_input],
            outputs=[college_address_output]
        )

# To run the Gradio app
demo.launch(debug=True, share=True)
