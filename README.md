# Oracle_Hcm.db
-- DIMENSION TABLE :LOCATION
CREATE TABLE dim_location (
location_id INT  PRIMARY KEY,
country VARCHAR(80) ,
state_name VARCHAR(80) ,
city VARCHAR(80) ,
office_type VARCHAR(50) 
 );
DROP TABLE dim_location;

-- DIMENSION TABLE : DEPARTMENT
CREATE TABLE dim_department (
department_id INT PRIMARY KEY,
department_name VARCHAR(100) UNIQUE,
business_unit VARCHAR(100),
cost_center_code VARCHAR(50) UNIQUE
 );

-- DIMENSION TABLE : JOB_ROLE 
CREATE TABLE dim_job_role (
job_role_id INT PRIMARY KEY,
job_title VARCHAR(100),
job_family VARCHAR(100),
job_level VARCHAR(50),
grade VARCHAR(50)
 );

-- DIMENSION TABLE : PAYROLL_ELEMENT
CREATE TABLE dim_payroll_element (
payroll_element_id INT PRIMARY KEY,
element_code VARCHAR(50) UNIQUE,
element_name VARCHAR(100),
element_type VARCHAR(50),
taxable_flag BOOLEAN
);

-- DIMENSION TABLE :CALENDER
CREATE TABLE dim_calendar (
calendar_date DATE PRIMARY KEY,
year INT,
quarter INT,
month_no INT,
month_name VARCHAR(20),
week_no INT,
day_name VARCHAR(20),
is_weekend BOOLEAN
);
--FACT TABLE: EMPLOYEE 
CREATE TABLE dim_employee (
    employee_id INT PRIMARY KEY,
    employee_name VARCHAR(100),
    date_of_birth VARCHAR(20),
    hire_date VARCHAR(20),
    gender VARCHAR(20),
    termination_date VARCHAR(20),
    employment_status VARCHAR(50),

    department_id INT,
    job_role_id INT,
    location_id INT,
    manager_id INT,

    employee_type VARCHAR(50),
    employee_code VARCHAR(50) UNIQUE,

    FOREIGN KEY (department_id)
        REFERENCES dim_department(department_id),

    FOREIGN KEY (job_role_id)
        REFERENCES dim_job_role(job_role_id),

    FOREIGN KEY (location_id)
        REFERENCES dim_location(location_id),

    FOREIGN KEY (manager_id)
        REFERENCES dim_employee(employee_id));

--FACT TABLE: fact_assignment_history
CREATE TABLE fact_assignment_history (
    assignment_id INT PRIMARY KEY,

    employee_id INT,

    old_department_id INT,
    new_department_id INT,

    old_job_role_id INT,
    new_job_role_id INT,

    assignment_start_date DATE,
    assignment_end_date DATE,
    assignment_status VARCHAR(50),
    assignment_type VARCHAR(50),
    transfer_reason VARCHAR(200),

    FOREIGN KEY (employee_id)
        REFERENCES dim_employee(employee_id),

    FOREIGN KEY (old_department_id)
        REFERENCES dim_department(department_id),

    FOREIGN KEY (new_department_id)
        REFERENCES dim_department(department_id),

    FOREIGN KEY (old_job_role_id)
        REFERENCES dim_job_role(job_role_id),

    FOREIGN KEY (new_job_role_id)
        REFERENCES dim_job_role(job_role_id)
);
--FACT TABLE: PAYROLL_RUN
CREATE TABLE fact_payroll_run (
    payroll_run_id INT PRIMARY KEY,

    employee_id INT,

    payroll_run_code VARCHAR(50) UNIQUE,
    payroll_month DATE,
    payroll_status VARCHAR(50),

    gross_pay NUMERIC(12,2),
	total_deductions INT,
	net_pay INT,
    employer_contribution NUMERIC(12,2),

    processed_date DATE,
    payment_date DATE,

    FOREIGN KEY (employee_id)
        REFERENCES dim_employee(employee_id));

--FACT TABLE:ELEMENT_ENTRY
CREATE TABLE fact_element_entry (
    element_entry_id INT PRIMARY KEY,

    employee_id INT,
    payroll_run_id INT,
    payroll_element_id INT,

    element_month DATE,
    element_amount NUMERIC(12,2),
    entry_status VARCHAR(50),

    FOREIGN KEY (employee_id)
        REFERENCES dim_employee(employee_id),

    FOREIGN KEY (payroll_run_id)
        REFERENCES fact_payroll_run(payroll_run_id),

    FOREIGN KEY (payroll_element_id)
        REFERENCES dim_payroll_element(payroll_element_id)
);
--FACT TABLE: ATTENDANCE_SUMMARY
CREATE TABLE fact_attendance_summary (
    attendance_id INT PRIMARY KEY,

    employee_id INT,

    attendance_month DATE,
    working_days INT,
    days_present INT,
    days_absent INT,
    leave_days INT,
    overtime_hours NUMERIC(8,2),

    FOREIGN KEY (employee_id)
        REFERENCES dim_employee(employee_id)
);
--FACT TABLE:ATTRITION
CREATE TABLE fact_attrition (
    attrition_id INT PRIMARY KEY,

    employee_id INT,

    attrition_date DATE,
    attrition_reason VARCHAR(200),
    exit_type VARCHAR(50),
    notice_period_days INT,
    eligible_for_rehire BOOLEAN,

    FOREIGN KEY (employee_id)
        REFERENCES dim_employee(employee_id)
);
