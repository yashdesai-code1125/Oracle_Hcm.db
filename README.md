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

/*
===============================================================================
ORACLE HCM WORKFORCE & PAYROLL ANALYTICS
PHASE 2 - REALISTIC DATA POPULATION
PostgreSQL

DATABASE:
    oracle_hcm_workforce_db

SCHEMA:
    oracle_hcm_workforce

RUN:
    1. Phase-1 DDL must already be completed successfully.
    2. Connect to oracle_hcm_workforce_db.
    3. Run this complete script.
    4. This script clears existing Phase-2 data in the project schema first.

DESIGN PRINCIPLES
-----------------
- Data is intentionally NOT evenly distributed.
- Employee hierarchy is realistic.
- Payroll values are driven by job grade / role.
- Attendance varies by employee type and month.
- Attrition is concentrated in selected employee groups.
- Assignment history contains realistic transfers/promotions.
- Payroll and element entries are relationally consistent.
===============================================================================
*/

SET search_path TO oracle_hcm_workforce;

DO $$
BEGIN
    IF NOT EXISTS (
        SELECT 1
        FROM information_schema.schemata
        WHERE schema_name = 'oracle_hcm_workforce'
    ) THEN
        RAISE EXCEPTION
        'Schema oracle_hcm_workforce does not exist. Run Phase-1 DDL first.';
    END IF;
END $$;

SELECT setseed(0.47291);


-- ============================================================================
-- 0. CLEAR EXISTING DATA
-- ============================================================================

TRUNCATE TABLE
    fact_attrition,
    fact_attendance_summary,
    fact_element_entry,
    fact_payroll_run,
    fact_assignment_history,
    dim_employee,
    dim_calendar,
    dim_payroll_element,
    dim_job_role,
    dim_department,
    dim_location
RESTART IDENTITY CASCADE;


-- ============================================================================
-- 1. LOCATIONS
-- Unequal workforce concentration by city.
-- ============================================================================

INSERT INTO dim_location
    (country, state_name, city, office_type)
VALUES
    ('India','Maharashtra','Pune','Corporate'),
    ('India','Maharashtra','Mumbai','Corporate'),
    ('India','Karnataka','Bengaluru','Corporate'),
    ('India','Telangana','Hyderabad','Technology'),
    ('India','Tamil Nadu','Chennai','Technology'),
    ('India','Delhi','New Delhi','Corporate'),
    ('India','Karnataka','Mysuru','Regional'),
    ('India','Gujarat','Ahmedabad','Regional'),
    ('India','West Bengal','Kolkata','Regional'),
    ('India','Uttar Pradesh','Noida','Technology'),
    ('India','Rajasthan','Jaipur','Regional'),
    ('India','Maharashtra','Nashik','Branch');


-- ============================================================================
-- 2. DEPARTMENTS
-- ============================================================================

INSERT INTO dim_department
    (department_name, business_unit, cost_center_code)
VALUES
    ('Human Resources','Corporate Services','CC-HR-001'),
    ('Finance','Corporate Services','CC-FIN-001'),
    ('Information Technology','Technology','CC-IT-001'),
    ('Engineering','Technology','CC-ENG-001'),
    ('Sales','Commercial','CC-SAL-001'),
    ('Marketing','Commercial','CC-MKT-001'),
    ('Operations','Operations','CC-OPS-001'),
    ('Customer Support','Operations','CC-CS-001'),
    ('Procurement','Corporate Services','CC-PRO-001'),
    ('Legal','Corporate Services','CC-LEG-001'),
    ('Analytics','Technology','CC-ANL-001'),
    ('Risk & Compliance','Corporate Services','CC-RSK-001'),
    ('Product Management','Technology','CC-PM-001'),
    ('Shared Services','Corporate Services','CC-SSC-001');


-- ============================================================================
-- 3. JOB ROLES
-- ============================================================================

INSERT INTO dim_job_role
    (job_title, job_family, job_level, grade)
VALUES
    ('HR Executive','Human Resources','Entry','G4'),
    ('HR Business Partner','Human Resources','Mid','G6'),
    ('HR Manager','Human Resources','Manager','G8'),
    ('Senior HR Manager','Human Resources','Senior Manager','G10'),

    ('Finance Analyst','Finance','Entry','G4'),
    ('Senior Finance Analyst','Finance','Mid','G6'),
    ('Finance Manager','Finance','Manager','G8'),
    ('Financial Controller','Finance','Senior Manager','G10'),

    ('Software Engineer','Technology','Entry','G5'),
    ('Senior Software Engineer','Technology','Mid','G7'),
    ('Technical Lead','Technology','Lead','G9'),
    ('Engineering Manager','Technology','Manager','G10'),
    ('Principal Engineer','Technology','Principal','G11'),

    ('Data Analyst','Analytics','Entry','G5'),
    ('Senior Data Analyst','Analytics','Mid','G7'),
    ('Data Scientist','Analytics','Mid','G8'),
    ('Analytics Manager','Analytics','Manager','G10'),

    ('Sales Executive','Sales','Entry','G4'),
    ('Senior Sales Executive','Sales','Mid','G6'),
    ('Sales Manager','Sales','Manager','G8'),
    ('Regional Sales Manager','Sales','Senior Manager','G10'),

    ('Marketing Executive','Marketing','Entry','G4'),
    ('Marketing Specialist','Marketing','Mid','G6'),
    ('Marketing Manager','Marketing','Manager','G8'),

    ('Operations Executive','Operations','Entry','G4'),
    ('Operations Analyst','Operations','Mid','G6'),
    ('Operations Manager','Operations','Manager','G8'),

    ('Customer Support Executive','Customer Support','Entry','G3'),
    ('Senior Support Executive','Customer Support','Mid','G5'),
    ('Support Manager','Customer Support','Manager','G8'),

    ('Procurement Executive','Procurement','Entry','G4'),
    ('Procurement Manager','Procurement','Manager','G8'),

    ('Legal Counsel','Legal','Mid','G7'),
    ('Senior Legal Counsel','Legal','Senior','G9'),

    ('Compliance Analyst','Risk','Entry','G5'),
    ('Risk Manager','Risk','Manager','G9'),

    ('Product Analyst','Product','Mid','G7'),
    ('Product Manager','Product','Manager','G9'),
    ('Senior Product Manager','Product','Senior Manager','G10'),

    ('Shared Services Executive','Shared Services','Entry','G3'),
    ('Shared Services Manager','Shared Services','Manager','G7');


-- ============================================================================
-- 4. PAYROLL ELEMENTS
-- ============================================================================

INSERT INTO dim_payroll_element
    (element_code, element_name, element_type, taxable_flag)
VALUES
    ('BASIC','Basic Salary','Earnings',TRUE),
    ('HRA','House Rent Allowance','Earnings',TRUE),
    ('SPECIAL','Special Allowance','Earnings',TRUE),
    ('BONUS','Performance Bonus','Earnings',TRUE),
    ('INCENTIVE','Sales Incentive','Earnings',TRUE),
    ('OVERTIME','Overtime Pay','Earnings',TRUE),
    ('PF','Employee Provident Fund','Deduction',FALSE),
    ('PT','Professional Tax','Deduction',FALSE),
    ('TDS','Income Tax','Deduction',FALSE),
    ('ESI','Employee State Insurance','Deduction',FALSE),
    ('LOAN','Salary Loan Deduction','Deduction',FALSE);


-- ============================================================================
-- 5. CALENDAR
-- ============================================================================

INSERT INTO dim_calendar
    (calendar_date, year, quarter, month_no, month_name,
     week_no, day_name, is_weekend)
SELECT
    d::DATE,
    EXTRACT(YEAR FROM d)::INT,
    EXTRACT(QUARTER FROM d)::INT,
    EXTRACT(MONTH FROM d)::INT,
    TO_CHAR(d,'Month'),
    EXTRACT(WEEK FROM d)::INT,
    TO_CHAR(d,'Day'),
    EXTRACT(ISODOW FROM d)::INT IN (6,7)
FROM generate_series(
    '2022-01-01'::DATE,
    '2026-12-31'::DATE,
    '1 day'
) d;


-- ============================================================================
-- 6. EMPLOYEES
--
-- 3,000 employees.
-- Department and location distributions are intentionally uneven.
-- Employees are inserted first without managers; managers are assigned after
-- insertion to avoid self-referencing insert problems.
-- ============================================================================

INSERT INTO dim_employee
    (employee_name, date_of_birth, hire_date, gender,
     termination_date, employment_status,
     department_id, job_role_id, location_id,
     manager_id, employee_type, employee_code)
SELECT
    (ARRAY[
        'Aarav','Vivaan','Aditya','Arjun','Rohan','Rahul','Kunal','Siddharth',
        'Aditi','Ananya','Priya','Sneha','Neha','Pooja','Kavya','Isha',
        'Nisha','Riya','Meera','Shreya','Tanvi','Sakshi','Mihir','Omkar'
    ])[1 + ((g-1)%24)]
    || ' ' ||
    (ARRAY[
        'Sharma','Patil','Kulkarni','Mehta','Shah','Joshi','Gupta','Verma',
        'Deshmukh','Rao','Nair','More','Singh','Iyer','Khan','Chavan'
    ])[1 + ((g*7-1)%16)]
    || ' ' || g,

    DATE '1972-01-01' + ((g*41)%10500),

    DATE '2014-01-01' + ((g*17)%4300),

    CASE
        WHEN g%10 < 6 THEN 'Male'
        ELSE 'Female'
    END,

    NULL,

    CASE
        WHEN g%100 < 91 THEN 'Active'
        WHEN g%100 < 96 THEN 'On Notice'
        ELSE 'Inactive'
    END,

    CASE
        WHEN g%100 < 12 THEN 3
        WHEN g%100 < 25 THEN 4
        WHEN g%100 < 40 THEN 7
        WHEN g%100 < 52 THEN 5
        WHEN g%100 < 61 THEN 8
        WHEN g%100 < 68 THEN 11
        WHEN g%100 < 74 THEN 13
        WHEN g%100 < 80 THEN 1
        WHEN g%100 < 85 THEN 2
        WHEN g%100 < 89 THEN 6
        WHEN g%100 < 93 THEN 9
        WHEN g%100 < 96 THEN 12
        WHEN g%100 < 98 THEN 10
        ELSE 14
    END,

    CASE
        WHEN g%100 < 25 THEN 9 + (g%5)
        WHEN g%100 < 40 THEN 14 + (g%5)
        WHEN g%100 < 52 THEN 18 + (g%3)
        WHEN g%100 < 68 THEN 24 + (g%4)
        WHEN g%100 < 80 THEN 28 + (g%4)
        ELSE 1 + (g%41)
    END,

    CASE
        WHEN g%100 < 30 THEN 1
        WHEN g%100 < 55 THEN 2
        WHEN g%100 < 72 THEN 3
        WHEN g%100 < 84 THEN 4
        WHEN g%100 < 91 THEN 5
        WHEN g%100 < 95 THEN 6
        WHEN g%100 < 98 THEN 8
        ELSE 10
    END,

    NULL,

    CASE
        WHEN g%100 < 65 THEN 'Permanent'
        WHEN g%100 < 82 THEN 'Contract'
        WHEN g%100 < 92 THEN 'Probation'
        WHEN g%100 < 97 THEN 'Intern'
        ELSE 'Part Time'
    END,

    'EMP' || LPAD(g::TEXT,6,'0')
FROM generate_series(1,3000) g;


-- ============================================================================
-- 7. EMPLOYEE MANAGER HIERARCHY
--
-- Manager is always an earlier employee ID and never the employee itself.
-- Senior employees are more likely to become managers.
-- ============================================================================

UPDATE dim_employee e
SET manager_id =
    CASE
        WHEN e.employee_id <= 20 THEN NULL
        WHEN e.employee_id % 17 = 0 THEN
            1 + ((e.employee_id*3)%20)
        WHEN e.employee_id % 9 = 0 THEN
            1 + ((e.employee_id*5)%80)
        WHEN e.employee_id % 5 = 0 THEN
            1 + ((e.employee_id*7)%250)
        ELSE
            1 + ((e.employee_id*11)%500)
    END
WHERE e.employee_id > 20;


-- ============================================================================
-- 8. TERMINATION DATES
--
-- Only a subset of inactive/on-notice employees are terminated.
-- Dates always occur after hire dates.
-- ============================================================================

UPDATE dim_employee
SET termination_date =
    hire_date + (
        450 + ((employee_id*37)%2200)
    ),
    employment_status = 'Terminated'
WHERE employee_id%100 >= 95
  AND employee_id > 20
  AND hire_date + (450 + ((employee_id*37)%2200)) <= DATE '2026-12-15';

UPDATE dim_employee
SET termination_date =
    DATE '2026-01-01' + ((employee_id*13)%330)
WHERE employment_status = 'On Notice'
  AND employee_id%7 = 0
  AND DATE '2026-01-01' + ((employee_id*13)%330) >= hire_date;


-- ============================================================================
-- 9. ASSIGNMENT HISTORY
--
-- Around 20% of employees receive historical assignment records.
-- Some are department transfers; some are promotions/role changes.
-- ============================================================================

INSERT INTO fact_assignment_history
    (employee_id,
     old_department_id, new_department_id,
     old_job_role_id, new_job_role_id,
     assignment_start_date, assignment_end_date,
     assignment_status, assignment_type, transfer_reason)
SELECT
    e.employee_id,

    CASE
        WHEN e.department_id = 1 THEN 3
        WHEN e.department_id = 3 THEN 7
        ELSE 1 + ((e.employee_id*3)%14)
    END,

    e.department_id,

    CASE
        WHEN e.job_role_id <= 4 THEN 5
        WHEN e.job_role_id BETWEEN 9 AND 13 THEN 9
        ELSE 1 + ((e.employee_id*7)%42)
    END,

    e.job_role_id,

    e.hire_date + (180 + (e.employee_id%900)),

    e.hire_date + (180 + (e.employee_id%900)) + (90 + (e.employee_id%700)),

    'Completed',

    CASE
        WHEN e.employee_id%5 IN (0,1) THEN 'Promotion'
        WHEN e.employee_id%5 = 2 THEN 'Transfer'
        WHEN e.employee_id%5 = 3 THEN 'Role Change'
        ELSE 'Internal Movement'
    END,

    CASE
        WHEN e.employee_id%5 IN (0,1) THEN 'Career progression'
        WHEN e.employee_id%5 = 2 THEN 'Business requirement'
        WHEN e.employee_id%5 = 3 THEN 'Skill alignment'
        ELSE 'Organizational restructuring'
    END
FROM dim_employee e
WHERE e.employee_id%5 = 0
  AND e.employee_id > 20
  AND e.hire_date + (180 + (e.employee_id%900)) < DATE '2025-01-01';


-- Additional current assignment for a smaller subset.
INSERT INTO fact_assignment_history
    (employee_id,
     old_department_id, new_department_id,
     old_job_role_id, new_job_role_id,
     assignment_start_date, assignment_end_date,
     assignment_status, assignment_type, transfer_reason)
SELECT
    e.employee_id,
    CASE WHEN e.department_id = 3 THEN 4 ELSE 3 END,
    e.department_id,
    CASE WHEN e.job_role_id >= 9 THEN 9 ELSE 5 END,
    e.job_role_id,
    DATE '2025-01-01' + (e.employee_id%300),
    NULL,
    'Active',
    CASE WHEN e.employee_id%2=0 THEN 'Promotion' ELSE 'Transfer' END,
    CASE WHEN e.employee_id%2=0 THEN 'Performance progression'
         ELSE 'Business requirement' END
FROM dim_employee e
WHERE e.employee_id%11 = 0
  AND e.employee_id > 50;


-- ============================================================================
-- 10. PAYROLL RUNS
--
-- 24 monthly payroll runs per employee = ~72,000 records.
-- Salary is driven by job grade.
-- Payroll status is not uniformly distributed.
-- ============================================================================

INSERT INTO fact_payroll_run
    (employee_id, payroll_run_code, payroll_month,
     payroll_status, gross_pay, total_deductions,
     net_pay, employer_contribution,
     processed_date, payment_date)
SELECT
    e.employee_id,

    'PR' || TO_CHAR(m,'YYYYMM') || LPAD(e.employee_id::TEXT,6,'0'),

    m::DATE,

    CASE
        WHEN e.employment_status = 'Terminated'
             AND m::DATE > COALESCE(e.termination_date, DATE '2099-01-01')
            THEN 'Cancelled'
        WHEN (e.employee_id + EXTRACT(MONTH FROM m)::INT)%97 = 0
            THEN 'On Hold'
        WHEN (e.employee_id + EXTRACT(MONTH FROM m)::INT)%53 = 0
            THEN 'Adjusted'
        ELSE 'Processed'
    END,

    ROUND((
        CASE
            WHEN jr.grade='G3' THEN 28000
            WHEN jr.grade='G4' THEN 38000
            WHEN jr.grade='G5' THEN 52000
            WHEN jr.grade='G6' THEN 68000
            WHEN jr.grade='G7' THEN 85000
            WHEN jr.grade='G8' THEN 110000
            WHEN jr.grade='G9' THEN 145000
            WHEN jr.grade='G10' THEN 185000
            WHEN jr.grade='G11' THEN 235000
            ELSE 50000
        END
        *
        CASE
            WHEN e.employee_type='Intern' THEN 0.55
            WHEN e.employee_type='Part Time' THEN 0.65
            WHEN e.employee_type='Contract' THEN 0.90
            WHEN e.employee_type='Probation' THEN 0.88
            ELSE 1.00
        END
        *
        CASE
            WHEN EXTRACT(MONTH FROM m)::INT IN (3,4) THEN 1.02
            WHEN EXTRACT(MONTH FROM m)::INT IN (10,11,12) THEN 1.03
            ELSE 1.00
        END
    )::NUMERIC,2),

    ROUND((
        (
            CASE
                WHEN jr.grade='G3' THEN 28000
                WHEN jr.grade='G4' THEN 38000
                WHEN jr.grade='G5' THEN 52000
                WHEN jr.grade='G6' THEN 68000
                WHEN jr.grade='G7' THEN 85000
                WHEN jr.grade='G8' THEN 110000
                WHEN jr.grade='G9' THEN 145000
                WHEN jr.grade='G10' THEN 185000
                WHEN jr.grade='G11' THEN 235000
                ELSE 50000
            END
            *
            CASE
                WHEN e.employee_type='Intern' THEN 0.55
                WHEN e.employee_type='Part Time' THEN 0.65
                WHEN e.employee_type='Contract' THEN 0.90
                WHEN e.employee_type='Probation' THEN 0.88
                ELSE 1.00
            END
        ) * 0.16
    )::NUMERIC,2),

    ROUND((
        (
            CASE
                WHEN jr.grade='G3' THEN 28000
                WHEN jr.grade='G4' THEN 38000
                WHEN jr.grade='G5' THEN 52000
                WHEN jr.grade='G6' THEN 68000
                WHEN jr.grade='G7' THEN 85000
                WHEN jr.grade='G8' THEN 110000
                WHEN jr.grade='G9' THEN 145000
                WHEN jr.grade='G10' THEN 185000
                WHEN jr.grade='G11' THEN 235000
                ELSE 50000
            END
            *
            CASE
                WHEN e.employee_type='Intern' THEN 0.55
                WHEN e.employee_type='Part Time' THEN 0.65
                WHEN e.employee_type='Contract' THEN 0.90
                WHEN e.employee_type='Probation' THEN 0.88
                ELSE 1.00
            END
        ) * 0.84
    )::NUMERIC,2),

    ROUND((
        (
            CASE
                WHEN jr.grade='G3' THEN 28000
                WHEN jr.grade='G4' THEN 38000
                WHEN jr.grade='G5' THEN 52000
                WHEN jr.grade='G6' THEN 68000
                WHEN jr.grade='G7' THEN 85000
                WHEN jr.grade='G8' THEN 110000
                WHEN jr.grade='G9' THEN 145000
                WHEN jr.grade='G10' THEN 185000
                WHEN jr.grade='G11' THEN 235000
                ELSE 50000
            END
            *
            CASE
                WHEN e.employee_type='Intern' THEN 0.55
                WHEN e.employee_type='Part Time' THEN 0.65
                WHEN e.employee_type='Contract' THEN 0.90
                WHEN e.employee_type='Probation' THEN 0.88
                ELSE 1.00
            END
        ) * 0.12
    )::NUMERIC,2),

    (m + INTERVAL '3 days')::DATE,

    CASE
        WHEN (e.employee_id + EXTRACT(MONTH FROM m)::INT)%97 = 0
            THEN (m + INTERVAL '8 days')::DATE
        ELSE (m + INTERVAL '5 days')::DATE
    END

FROM dim_employee e
JOIN dim_job_role jr
  ON jr.job_role_id = e.job_role_id
CROSS JOIN generate_series(
    DATE '2024-01-01',
    DATE '2025-12-01',
    INTERVAL '1 month'
) m
WHERE
    e.hire_date <= (m + INTERVAL '1 month - 1 day')::DATE
    AND (
        e.termination_date IS NULL
        OR m::DATE <= DATE_TRUNC('month',e.termination_date)::DATE
    );


-- ============================================================================
-- 11. PAYROLL ELEMENT ENTRIES
--
-- Earnings and deductions are derived from the payroll run.
-- Different employees receive different combinations of elements.
-- ============================================================================

INSERT INTO fact_element_entry
    (employee_id, payroll_run_id, payroll_element_id,
     element_month, element_amount, entry_status)
SELECT
    p.employee_id,
    p.payroll_run_id,
    pe.payroll_element_id,
    p.payroll_month,

    ROUND((
        CASE pe.element_code
            WHEN 'BASIC' THEN p.gross_pay * 0.45
            WHEN 'HRA' THEN p.gross_pay * 0.20
            WHEN 'SPECIAL' THEN p.gross_pay * 0.35
            WHEN 'BONUS' THEN
                CASE
                    WHEN EXTRACT(MONTH FROM p.payroll_month)::INT IN (3,10,11,12)
                    THEN p.gross_pay * 0.08
                    ELSE 0
                END
            WHEN 'INCENTIVE' THEN
                CASE
                    WHEN e.department_id IN (5,6)
                    THEN p.gross_pay * (0.02 + ((e.employee_id%5)/100.0))
                    ELSE 0
                END
            WHEN 'OVERTIME' THEN
                CASE
                    WHEN e.employee_id%6=0 THEN 2500 + (e.employee_id%1800)
                    WHEN e.employee_id%13=0 THEN 1200 + (e.employee_id%900)
                    ELSE 0
                END
            WHEN 'PF' THEN p.gross_pay * 0.06
            WHEN 'PT' THEN
                CASE
                    WHEN p.gross_pay >= 75000 THEN 200
                    WHEN p.gross_pay >= 30000 THEN 150
                    ELSE 0
                END
            WHEN 'TDS' THEN
                CASE
                    WHEN p.gross_pay >= 150000 THEN p.gross_pay * 0.12
                    WHEN p.gross_pay >= 90000 THEN p.gross_pay * 0.07
                    WHEN p.gross_pay >= 60000 THEN p.gross_pay * 0.03
                    ELSE 0
                END
            WHEN 'ESI' THEN
                CASE
                    WHEN p.gross_pay <= 50000 THEN p.gross_pay * 0.0075
                    ELSE 0
                END
            WHEN 'LOAN' THEN
                CASE
                    WHEN e.employee_id%19=0 THEN 3500 + (e.employee_id%2500)
                    ELSE 0
                END
            ELSE 0
        END
    )::NUMERIC,2),

    CASE
        WHEN pe.element_code IN ('BONUS','INCENTIVE','OVERTIME')
             AND (
                 CASE pe.element_code
                     WHEN 'BONUS' THEN p.gross_pay * 0.08
                     WHEN 'INCENTIVE' THEN
                         CASE WHEN e.department_id IN (5,6)
                              THEN p.gross_pay * 0.03 ELSE 0 END
                     WHEN 'OVERTIME' THEN
                         CASE WHEN e.employee_id%6=0 THEN 2500 ELSE 0 END
                     ELSE 0
                 END
             ) > 0
            THEN 'Processed'
        ELSE 'Processed'
    END
FROM fact_payroll_run p
JOIN dim_employee e
  ON e.employee_id = p.employee_id
JOIN dim_payroll_element pe
  ON (
      pe.element_code IN ('BASIC','HRA','SPECIAL','PF','PT','TDS')
      OR (pe.element_code='BONUS' AND EXTRACT(MONTH FROM p.payroll_month)::INT IN (3,10,11,12))
      OR (pe.element_code='INCENTIVE' AND e.department_id IN (5,6))
      OR (pe.element_code='OVERTIME' AND e.employee_id%6=0)
      OR (pe.element_code='ESI' AND p.gross_pay <= 50000)
      OR (pe.element_code='LOAN' AND e.employee_id%19=0)
  )
WHERE p.payroll_status <> 'Cancelled';


-- ============================================================================
-- 12. ATTENDANCE SUMMARY
--
-- Business rule:
-- days_present + days_absent + leave_days = working_days.
-- This guarantees the Phase-1 chk_attendance_days constraint is satisfied.
--
-- 24 monthly summaries per eligible employee.
-- Attendance varies by employee type and season.
-- ============================================================================

INSERT INTO fact_attendance_summary
    (employee_id, attendance_month,
     working_days, days_present, days_absent,
     leave_days, overtime_hours)
SELECT
    e.employee_id,
    m::DATE,

    /* Working days */
    (
        21 + ((EXTRACT(MONTH FROM m)::INT + e.employee_id) % 3)
    ),

    /* Days present = working days - absence - leave */
    (
        21 + ((EXTRACT(MONTH FROM m)::INT + e.employee_id) % 3)
    )
    -
    CASE
        WHEN e.employee_id % 17 = 0 THEN 2
        WHEN e.employee_id % 9 = 0 THEN 1
        WHEN e.employee_id % 23 = 0 THEN 3
        ELSE 0
    END
    -
    CASE
        WHEN e.employee_id % 17 = 0 THEN 2
        WHEN e.employee_id % 11 = 0 THEN 1
        ELSE 1 + ((e.employee_id + EXTRACT(MONTH FROM m)::INT) % 2)
    END,

    /* Days absent */
    CASE
        WHEN e.employee_id % 17 = 0 THEN 2
        WHEN e.employee_id % 9 = 0 THEN 1
        WHEN e.employee_id % 23 = 0 THEN 3
        ELSE 0
    END,

    /* Leave days */
    CASE
        WHEN e.employee_id % 17 = 0 THEN 2
        WHEN e.employee_id % 11 = 0 THEN 1
        ELSE 1 + ((e.employee_id + EXTRACT(MONTH FROM m)::INT) % 2)
    END,

    /* Overtime */
    ROUND((
        CASE
            WHEN e.department_id IN (3,4,11,13)
                 AND e.employee_id % 5 = 0
                THEN 8 + ((e.employee_id + EXTRACT(MONTH FROM m)::INT) % 18)
            WHEN e.department_id IN (7,8)
                 AND e.employee_id % 7 = 0
                THEN 5 + ((e.employee_id + EXTRACT(MONTH FROM m)::INT) % 12)
            ELSE
                (e.employee_id + EXTRACT(MONTH FROM m)::INT) % 6
        END
    )::NUMERIC,2)

FROM dim_employee e
CROSS JOIN generate_series(
    DATE '2024-01-01',
    DATE '2025-12-01',
    INTERVAL '1 month'
) m
WHERE e.hire_date <= (m + INTERVAL '1 month - 1 day')::DATE
  AND (
      e.termination_date IS NULL
      OR m::DATE <= DATE_TRUNC('month',e.termination_date)::DATE
  );


-- ============================================================================
-- 13. ATTRITION
--
-- Attrition is a subset of terminated employees.
-- Voluntary exits are more common than involuntary exits.
-- ============================================================================

INSERT INTO fact_attrition
    (employee_id, attrition_date, attrition_reason,
     exit_type, notice_period_days, eligible_for_rehire)
SELECT
    e.employee_id,
    e.termination_date,

    CASE
        WHEN e.employee_id%10 IN (0,1,2,3,4)
            THEN 'Better career opportunity'
        WHEN e.employee_id%10 IN (5,6)
            THEN 'Compensation'
        WHEN e.employee_id%10 = 7
            THEN 'Relocation'
        WHEN e.employee_id%10 = 8
            THEN 'Personal reasons'
        ELSE
            'Performance / Policy'
    END,

    CASE
        WHEN e.employee_id%10 < 8 THEN 'Voluntary'
        ELSE 'Involuntary'
    END,

    CASE
        WHEN e.employee_id%10 < 7 THEN 30
        WHEN e.employee_id%10 IN (7,8) THEN 60
        ELSE 15
    END,

    CASE
        WHEN e.employee_id%10 IN (0,1,2,7) THEN TRUE
        ELSE FALSE
    END

FROM dim_employee e
WHERE e.employment_status = 'Terminated'
  AND e.termination_date IS NOT NULL;


-- ============================================================================
-- 14. VALIDATION - RECORD COUNTS
-- ============================================================================

SELECT 'dim_location' AS table_name, COUNT(*) AS record_count
FROM dim_location
UNION ALL
SELECT 'dim_department', COUNT(*) FROM dim_department
UNION ALL
SELECT 'dim_job_role', COUNT(*) FROM dim_job_role
UNION ALL
SELECT 'dim_payroll_element', COUNT(*) FROM dim_payroll_element
UNION ALL
SELECT 'dim_calendar', COUNT(*) FROM dim_calendar
UNION ALL
SELECT 'dim_employee', COUNT(*) FROM dim_employee
UNION ALL
SELECT 'fact_assignment_history', COUNT(*) FROM fact_assignment_history
UNION ALL
SELECT 'fact_payroll_run', COUNT(*) FROM fact_payroll_run
UNION ALL
SELECT 'fact_element_entry', COUNT(*) FROM fact_element_entry
UNION ALL
SELECT 'fact_attendance_summary', COUNT(*) FROM fact_attendance_summary
UNION ALL
SELECT 'fact_attrition', COUNT(*) FROM fact_attrition
ORDER BY table_name;


-- ============================================================================
-- 15. BUSINESS VALIDATION QUERIES
-- ============================================================================

-- Employee distribution by department
SELECT
    d.department_name,
    COUNT(e.employee_id) AS employee_count,
    ROUND(
        COUNT(e.employee_id) * 100.0 /
        SUM(COUNT(e.employee_id)) OVER (),2
    ) AS workforce_pct
FROM dim_department d
LEFT JOIN dim_employee e
    ON e.department_id = d.department_id
GROUP BY d.department_name
ORDER BY employee_count DESC;


-- Workforce by employment status
SELECT
    employment_status,
    COUNT(*) AS employee_count
FROM dim_employee
GROUP BY employment_status
ORDER BY employee_count DESC;


-- Payroll by department
SELECT
    d.department_name,
    COUNT(p.payroll_run_id) AS payroll_records,
    ROUND(SUM(p.gross_pay),2) AS gross_pay,
    ROUND(SUM(p.total_deductions),2) AS deductions,
    ROUND(SUM(p.net_pay),2) AS net_pay
FROM fact_payroll_run p
JOIN dim_employee e
    ON e.employee_id = p.employee_id
JOIN dim_department d
    ON d.department_id = e.department_id
WHERE p.payroll_status <> 'Cancelled'
GROUP BY d.department_name
ORDER BY gross_pay DESC;


-- Attendance performance
SELECT
    d.department_name,
    ROUND(AVG(a.days_present),2) AS avg_days_present,
    ROUND(AVG(a.days_absent),2) AS avg_days_absent,
    ROUND(AVG(a.leave_days),2) AS avg_leave_days,
    ROUND(SUM(a.overtime_hours),2) AS total_overtime_hours
FROM fact_attendance_summary a
JOIN dim_employee e
    ON e.employee_id = a.employee_id
JOIN dim_department d
    ON d.department_id = e.department_id
GROUP BY d.department_name
ORDER BY total_overtime_hours DESC;


-- Attrition by reason
SELECT
    attrition_reason,
    exit_type,
    COUNT(*) AS attrition_count
FROM fact_attrition
GROUP BY attrition_reason, exit_type
ORDER BY attrition_count DESC;


-- Assignment movement
SELECT
    assignment_type,
    assignment_status,
    COUNT(*) AS assignment_count
FROM fact_assignment_history
GROUP BY assignment_type, assignment_status
ORDER BY assignment_count DESC;


-- Payroll consistency check
SELECT COUNT(*) AS payroll_records_with_incorrect_net_pay
FROM fact_payroll_run
WHERE payroll_status <> 'Cancelled'
  AND ABS(net_pay - (gross_pay - total_deductions)) > 0.05;


-- Employee hierarchy validation
SELECT COUNT(*) AS invalid_manager_links
FROM dim_employee e
JOIN dim_employee m
    ON e.manager_id = m.employee_id
WHERE e.manager_id = e.employee_id;


-- Attendance consistency validation
SELECT COUNT(*) AS invalid_attendance_records
FROM fact_attendance_summary
WHERE days_present + days_absent + leave_days > working_days;


-- Attrition consistency validation
SELECT COUNT(*) AS invalid_attrition_records
FROM fact_attrition a
JOIN dim_employee e
    ON e.employee_id = a.employee_id
WHERE a.attrition_date < e.hire_date;


-- ============================================================================
-- END OF PHASE 2
-- ============================================================================
--===============================================================================
SELECT
e.employee_code,
e.employee_name,
e.gender,
d.department_name AS department,
j.job_title,
j.job_level,
l.office_type AS location,
l.city,
e.employee_type,
e.employment_status,
e.hire_date
FROM dim_employee e
JOIN dim_department d
    ON e.department_id = d.department_id
JOIN dim_job_role j
    ON e.job_role_id = j.job_role_id
JOIN dim_location l
    ON e.location_id = l.location_id
ORDER BY e.hire_date ASC;
--======================================================================================================
--- question num  2----
SELECT
e.employee_name AS employee_name,
ed.department_name AS employee_department,
ej.job_title AS employee_job_title,

m.employee_name AS manager_name,
mj.job_title AS manager_job_title,
md.department_name AS manager_department

FROM dim_employee e

LEFT JOIN dim_employee m
ON e.manager_id = m.employee_id

LEFT JOIN dim_department ed
ON e.department_id = ed.department_id

LEFT JOIN dim_job_role ej
ON e.job_role_id = ej.job_role_id

LEFT JOIN dim_department md
ON m.department_id = md.department_id

LEFT JOIN dim_job_role mj
ON m.job_role_id = mj.job_role_id;

-- Question num 3-----
SELECT d.department_name AS department,

COUNT(e.employee_id) AS total_employees,

COUNT(
    CASE
WHEN e.employment_status = 'Active'
    THEN e.employee_id
END
) AS active_employees,

COUNT(
CASE
WHEN e.employment_status = 'Terminated'THEN e.employee_id

END
) AS terminated_employees,

COUNT(
CASE
WHEN e.employee_type = 'Permanent' THEN e.employee_id
END
) AS permanent_employees,

COUNT(
CASE
WHEN e.employee_type = 'Contract' THEN e.employee_id
 END
) AS contract_employees,

ROUND(
COUNT(e.employee_id) * 100.0 /SUM(COUNT(e.employee_id)) OVER (), 2
 ) AS workforce_percentage

FROM dim_department d

LEFT JOIN dim_employee e
ON d.department_id = e.department_id

GROUP BY d.department_id, d.department_name

ORDER BY total_employees DESC;
---- Question num 4 ---
--select * from fact_payroll_run;
--select * from dim_employee;

WITH latest_payroll AS (
SELECT
employee_id,
gross_pay,
ROW_NUMBER() OVER (PARTITION BY employee_id ORDER BY payroll_month DESC
) AS rn
    FROM fact_payroll_run
)

SELECT
e.employee_name AS employee,
d.department_name AS department,
j.grade AS job_grade,
p.gross_pay,

CASE
WHEN p.gross_pay < 40000 THEN 'Entry'
WHEN p.gross_pay <= 75000 THEN 'Junior'
WHEN p.gross_pay <= 125000 THEN 'Mid-Level'
WHEN p.gross_pay <= 200000 THEN 'Senior'
ELSE 'Leadership'
END AS salary_band

FROM dim_employee e

JOIN dim_department d
ON e.department_id = d.department_id

JOIN dim_job_role j
ON e.job_role_id = j.job_role_id

JOIN latest_payroll p
ON e.employee_id = p.employee_id
 AND p.rn = 1;
--- question no 5 --
SELECT employee_name AS Employee ,hire_date AS "Hire Date",

EXTRACT(YEAR FROM AGE(CURRENT_DATE, hire_date)) AS "Years of Service",

CASE
WHEN AGE(CURRENT_DATE, hire_date) < INTERVAL '2 years' THEN 'New'
WHEN AGE(CURRENT_DATE, hire_date) < INTERVAL '5 years' THEN 'Experienced'
WHEN AGE(CURRENT_DATE, hire_date) < INTERVAL '10 years'THEN 'Senior'

ELSE 'Long Tenure'
    END AS "Tenure Category"
FROM dim_employee;
select * from dim_employee;
commit;
--- question numm 6--
SELECT
    payroll_month AS payroll_month,
COUNT(DISTINCT employee_id) AS number_of_employees_paid,

SUM(gross_pay) AS total_gross_pay,
SUM(employer_contribution) AS total_employer_contribution

FROM fact_payroll_run
GROUP BY payroll_month
ORDER BY payroll_month ASC;
-------- question num7---

SELECT
    EXTRACT(YEAR FROM hire_date) AS "Year",
    COUNT(*) AS "Number of Employees Hired"
FROM dim_employee
GROUP BY EXTRACT(YEAR FROM hire_date)
ORDER BY "Year";
 --find highest hiring year--
 SELECT
    EXTRACT(YEAR FROM hire_date) AS "Year",
    COUNT(*) AS "Number of Employees Hired"
FROM dim_employee
GROUP BY EXTRACT(YEAR FROM hire_date)
ORDER BY COUNT(*) DESC
LIMIT 1;
--- question num 8---
--claculate age--
SELECT
    CASE
        WHEN EXTRACT(YEAR FROM AGE(CURRENT_DATE, e.date_of_birth)) < 25 THEN '< 25'
        WHEN EXTRACT(YEAR FROM AGE(CURRENT_DATE, e.date_of_birth)) BETWEEN 25 AND 34 THEN '25–34'
        WHEN EXTRACT(YEAR FROM AGE(CURRENT_DATE, e.date_of_birth)) BETWEEN 35 AND 44 THEN '35–44'
        WHEN EXTRACT(YEAR FROM AGE(CURRENT_DATE, e.date_of_birth)) BETWEEN 45 AND 54 THEN '45–54'

     ELSE '55+'
END AS "Age Group",

COUNT(DISTINCT e.employee_id) AS "Employee Count",

ROUND(AVG(p.gross_pay), 2) AS "Average Gross Pay"

FROM dim_employee e
JOIN fact_payroll_run p
    ON e.employee_id = p.employee_id
GROUP BY
    CASE
       WHEN EXTRACT(YEAR FROM AGE(CURRENT_DATE, e.date_of_birth)) < 25 THEN '< 25'
       WHEN EXTRACT(YEAR FROM AGE(CURRENT_DATE, e.date_of_birth)) BETWEEN 25 AND 34 THEN '25–34'
       WHEN EXTRACT(YEAR FROM AGE(CURRENT_DATE, e.date_of_birth)) BETWEEN 35 AND 44 THEN '35–44'
       WHEN EXTRACT(YEAR FROM AGE(CURRENT_DATE, e.date_of_birth)) BETWEEN 45 AND 54 THEN '45–54'
  ELSE '55+'
END
ORDER BY "Age Group";
--question num9---

SELECT
TO_CHAR(attrition_date, 'YYYY-MM') AS month,

COUNT(*) AS attrition_count,

COUNT(CASE WHEN exit_type = 'Voluntary' THEN 1
   END ) AS voluntary_exits,

COUNT( CASE WHEN exit_type = 'Involuntary' THEN 1
END
    ) AS involuntary_exits,

ROUND(AVG(notice_period_days), 2) AS average_notice_period

FROM fact_attrition

GROUP BY TO_CHAR(attrition_date, 'YYYY-MM')

ORDER BY month ASC;
--month with the highest attrition
SELECT
    TO_CHAR(attrition_date, 'YYYY-MM') AS month,
    COUNT(*) AS attrition_count
FROM fact_attrition
GROUP BY TO_CHAR(attrition_date, 'YYYY-MM')
ORDER BY attrition_count DESC
LIMIT 1;
-- question 10--
WITH employee_payroll AS (
    SELECT
        employee_id,

COUNT(*) AS total_payroll_records,
ROUND(AVG(gross_pay), 2) AS average_gross_pay,
ROUND(AVG(total_deductions), 2) AS average_deduction,
ROUND(AVG(net_pay), 2) AS average_net_pay,
SUM(employer_contribution) AS total_employer_contribution

FROM fact_payroll_run
GROUP BY employee_id
)

SELECT
    employee_id,
    total_payroll_records,
    average_gross_pay,
    average_deduction,
    average_net_pay,
    total_employer_contribution

FROM employee_payroll
WHERE average_gross_pay > (
    SELECT AVG(average_gross_pay)
    FROM employee_payroll
);
select* from employee_payroll;
--question11---

WITH department_payroll AS (
    SELECT
        d.department_id,
        d.department_name AS department,

COUNT(DISTINCT p.employee_id) AS employee_count,

SUM(p.gross_pay) AS total_gross_pay,

SUM(p.net_pay) AS total_net_pay,

ROUND(AVG(p.gross_pay), 2) AS average_gross_pay

FROM dim_department d

JOIN dim_employee e
ON d.department_id = e.department_id

JOIN fact_payroll_run p
ON e.employee_id = p.employee_id

GROUP BY
        d.department_id,
        d.department_name
),
ranked_departments AS (
    SELECT
        department,
        employee_count,
        total_gross_pay,
        total_net_pay,
        average_gross_pay,

        RANK() OVER (
            ORDER BY total_gross_pay DESC
        ) AS payroll_rank

    FROM department_payroll
)

SELECT
    department,
    employee_count,
    total_gross_pay,
    total_net_pay,
    average_gross_pay,
    payroll_rank

FROM ranked_departments
WHERE payroll_rank <= 5
ORDER BY payroll_rank;
--question num 12--
WITH department_attendance AS (
    SELECT
        d.department_name AS department,
        SUM(a.working_days) AS total_working_days,
        SUM(a.days_present) AS total_days_present,
        SUM(a.days_absent) AS total_days_absent,
        SUM(a.leave_days) AS total_leave_days,
        SUM(a.overtime_hours) AS total_overtime_hours
FROM dim_department d
JOIN dim_employee e
ON d.department_id = e.department_id
JOIN fact_attendance_summary a
ON e.employee_id = a.employee_id
GROUP BY
        d.department_id,
        d.department_name
)

SELECT
department,
total_working_days,
total_days_present,
total_days_absent,
total_leave_days,
total_overtime_hours,
ROUND(
total_days_present * 100.0 /
NULLIF(total_working_days, 0),
        2
    ) AS attendance_rate
FROM department_attendance
WHERE
    total_days_present * 100.0 /
    NULLIF(total_working_days, 0) >90
ORDER BY attendance_rate ASC;
-- question 13--
WITH employee_count AS (
    SELECT
d.department_id,
d.department_name AS department,
COUNT(e.employee_id) AS total_employees
FROM dim_department d
LEFT JOIN dim_employee e
        ON d.department_id = e.department_id
GROUP BY d.department_id, d.department_name
),

terminated_count AS (
    SELECT
e.department_id,
COUNT(DISTINCT a.employee_id) AS terminated_employees
FROM dim_employee e
JOIN fact_attrition a
ON e.employee_id = a.employee_id
WHERE a.exit_type = 'Involuntary' OR a.exit_type = 'Voluntary'
GROUP BY e.department_id
),
attrition_rate AS (
SELECT
        ec.department,
        ec.total_employees,
COALESCE(tc.terminated_employees, 0) AS terminated_employees,

ROUND(
COALESCE(tc.terminated_employees, 0) * 100.0 / NULLIF(ec.total_employees, 0),2
) AS attrition_rate

FROM employee_count ec
LEFT JOIN terminated_count tc
ON ec.department_id = tc.department_id
)

SELECT
    department,
    total_employees,
    terminated_employees,
    attrition_rate
FROM attrition_rate
WHERE attrition_rate > 5
ORDER BY attrition_rate DESC;
--question14--
WITH employee_overtime AS (
SELECT
e.employee_id,
e.employee_code,
e.employee_name,
d.department_name AS department,
j.job_title AS job_role,

SUM(a.overtime_hours) AS total_overtime_hours,

ROUND(
            AVG(a.overtime_hours),
            2
        ) AS average_monthly_overtime

    FROM dim_employee e

    JOIN dim_department d
        ON e.department_id = d.department_id

    JOIN dim_job_role j
        ON e.job_role_id = j.job_role_id

    JOIN fact_attendance_summary a
        ON e.employee_id = a.employee_id

    GROUP BY
        e.employee_id,
        e.employee_code,
        e.employee_name,
        d.department_name,
        j.job_title
),

overall_average AS (
    SELECT
        AVG(total_overtime_hours) AS overall_employee_average
    FROM employee_overtime
)

SELECT
    eo.employee_code,
    eo.employee_name,
    eo.department,
    eo.job_role,
    eo.total_overtime_hours,
    eo.average_monthly_overtime

FROM employee_overtime eo

CROSS JOIN overall_average oa

WHERE eo.total_overtime_hours > oa.overall_employee_average

ORDER BY eo.total_overtime_hours DESC

LIMIT 20;
select * from dim_job_role;
-- question 16--
CREATE OR REPLACE VIEW vw_employee_payroll AS

SELECT
    e.employee_name AS employee,
    d.department_name AS department,
    p.payroll_month,
    p.gross_pay,
    p.total_deductions,
    p.net_pay,
    p.employer_contribution,
    p.payroll_status,
    p.payment_date

FROM dim_employee e

JOIN dim_department d
    ON e.department_id = d.department_id

JOIN fact_payroll_run p
    ON e.employee_id = p.employee_id;
	select * from vw_employee_payroll;
--question 17--
CREATE OR REPLACE VIEW vw_department_hr_kpi AS

SELECT
    d.department_name AS department,

    COUNT(DISTINCT e.employee_id) AS employee_count,

    COUNT(DISTINCT CASE
        WHEN e.employment_status = 'Active'
        THEN e.employee_id
    END) AS active_employees,

    COUNT(DISTINCT CASE
        WHEN e.employment_status = 'Terminated'
        THEN e.employee_id
    END) AS terminated_employees,

    ROUND(AVG(p.gross_pay), 2) AS average_gross_pay,

    SUM(p.gross_pay) AS total_payroll_cost,

    SUM(a.overtime_hours) AS total_overtime_hours,

    COUNT(DISTINCT at.employee_id) AS attrition_count,

    ROUND(
        COUNT(DISTINCT at.employee_id) * 100.0
        / NULLIF(COUNT(DISTINCT e.employee_id), 0),
        2
    ) AS attrition_rate

FROM dim_department d

LEFT JOIN dim_employee e
    ON d.department_id = e.department_id

LEFT JOIN fact_payroll_run p
    ON e.employee_id = p.employee_id

LEFT JOIN fact_attendance_summary a
    ON e.employee_id = a.employee_id

LEFT JOIN fact_attrition at
    ON e.employee_id = at.employee_id

GROUP BY
    d.department_id,
    d.department_name;

---question 18---
SELECT
    employee,
    department,
    ROUND(AVG(gross_pay), 2) AS average_gross_pay,
    ROUND(AVG(net_pay), 2) AS average_net_pay,
    COUNT(*) AS total_payroll_records
FROM vw_employee_payroll
GROUP BY employee, department
ORDER BY average_gross_pay DESC
LIMIT 20;
--question 19--
CREATE OR REPLACE FUNCTION fn_employee_payroll_summary(p_employee_id INT)
RETURNS TABLE (
    employee_name VARCHAR,
    total_payroll_records BIGINT,
    average_gross_pay NUMERIC,
    average_net_pay NUMERIC,
    total_deductions NUMERIC
)
LANGUAGE plpgsql
AS $$
BEGIN

    RETURN QUERY
    SELECT
        e.employee_name,
        COUNT(p.employee_id),
        ROUND(AVG(p.gross_pay), 2),
        ROUND(AVG(p.net_pay), 2),
        SUM(p.total_deductions)

    FROM dim_employee e

    JOIN fact_payroll_run p
        ON e.employee_id = p.employee_id

    WHERE e.employee_id = p_employee_id

    GROUP BY e.employee_name;

END;
$$;
SELECT * FROM fn_employee_payroll_summary(1);

SELECT * FROM fn_employee_payroll_summary(2);

SELECT * FROM fn_employee_payroll_summary(3);

SELECT * FROM fn_employee_payroll_summary(4);

SELECT * FROM fn_employee_payroll_summary(5);
---question 20----
CREATE OR REPLACE FUNCTION fn_salary_band(gross_pay NUMERIC)
RETURNS VARCHAR
LANGUAGE plpgsql
AS $$
BEGIN

    IF gross_pay < 40000 THEN
        RETURN 'Entry';

    ELSIF gross_pay <= 75000 THEN
        RETURN 'Junior';

    ELSIF gross_pay <= 125000 THEN
        RETURN 'Mid-Level';

    ELSIF gross_pay <= 200000 THEN
        RETURN 'Senior';

    ELSE
        RETURN 'Leadership';
    END IF;

END;
$$;
--- test  salary value---
SELECT
    35000 AS gross_pay,
    fn_salary_band(35000) AS salary_band

UNION ALL

SELECT
    60000,
    fn_salary_band(60000)

UNION ALL

SELECT
    100000,
    fn_salary_band(100000)

UNION ALL

SELECT
    150000,
    fn_salary_band(150000)

UNION ALL

SELECT
    250000,
    fn_salary_band(250000);
-- test  using actual dataset values 
	SELECT
    gross_pay,
    fn_salary_band(gross_pay) AS salary_band
FROM fact_payroll_run
LIMIT 10;
--question num 21--
CREATE OR REPLACE FUNCTION fn_employee_tenure(hire_date DATE)
RETURNS TABLE (
    years_of_service NUMERIC,
    tenure_category VARCHAR
)
LANGUAGE plpgsql
AS $$
BEGIN

    years_of_service :=
        ROUND(
            (CURRENT_DATE - hire_date) / 365.25,
            1
        );

    IF years_of_service < 2 THEN
        tenure_category := 'New';

    ELSIF years_of_service <= 5 THEN
        tenure_category := 'Experienced';

    ELSIF years_of_service <= 10 THEN
        tenure_category := 'Senior';

    ELSE
        tenure_category := 'Long Tenure';
    END IF;

    RETURN NEXT;

END;
$$;
---Test with actual employee hire date--
SELECT
    e.employee_name AS employee,
    e.hire_date,
    t.years_of_service,
    t.tenure_category
FROM dim_employee e
CROSS JOIN LATERAL
    fn_employee_tenure(e.hire_date) t
LIMIT 10;
-- question num 22--
-- create procedure--

CREATE OR REPLACE PROCEDURE sp_update_employee_status()
LANGUAGE plpgsql
AS $$
BEGIN

    UPDATE dim_employee
    SET employment_status = 'Terminated'
    WHERE termination_date IS NOT NULL
      AND termination_date <= CURRENT_DATE
      AND employment_status <> 'Terminated';

END;
$$;
---call  procedure
CALL sp_update_employee_status();
-- Check the result--
SELECT
    employee_id,
    employee_name,
    termination_date,
    employment_status
FROM dim_employee
WHERE termination_date IS NOT NULL;
--- question num 23--
CREATE OR REPLACE PROCEDURE sp_apply_payroll_adjustment(
    p_employee_id INT,
    p_payroll_month DATE,
    p_adjustment_amount NUMERIC
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_gross_pay NUMERIC;
    v_deductions NUMERIC;
BEGIN

    -- Find the payroll record
    SELECT gross_pay, total_deductions
    INTO v_gross_pay, v_deductions
    FROM fact_payroll_run
    WHERE employee_id = p_employee_id
      AND payroll_month = p_payroll_month;

    -- If record does not exist
    IF NOT FOUND THEN
        RAISE EXCEPTION
        'Payroll record not found for employee % and month %',
        p_employee_id, p_payroll_month;
    END IF;

    -- Add adjustment to gross pay
    v_gross_pay := v_gross_pay + p_adjustment_amount;

    -- Recalculate deductions
    v_deductions := v_gross_pay * 0.10;

    -- Recalculate and update payroll
    UPDATE fact_payroll_run
    SET
        gross_pay = v_gross_pay,
        total_deductions = v_deductions,
        net_pay = v_gross_pay - v_deductions
    WHERE employee_id = p_employee_id
      AND payroll_month = p_payroll_month;

END;
$$;
----call--
CALL sp_apply_payroll_adjustment(
    1,
    '2024-01-01',
    5000
);
SELECT
    employee_id,
    payroll_month,
    gross_pay,
    total_deductions,
    net_pay
FROM fact_payroll_run
WHERE employee_id = 1
  AND payroll_month = '2024-01-01';
----question num 24--
-- crate the procedure--
CREATE OR REPLACE PROCEDURE sp_record_employee_transfer(
    p_employee_id INT,
    p_new_department_id INT,
    p_new_job_role_id INT,
    p_assignment_start_date DATE,
    p_transfer_reason VARCHAR
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_old_department_id INT;
    v_old_job_role_id INT;
BEGIN

    -- 1. Capture existing department and job role
    SELECT
        department_id,
        job_role_id
    INTO
        v_old_department_id,
        v_old_job_role_id
    FROM dim_employee
    WHERE employee_id = p_employee_id;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'Employee % not found', p_employee_id;
    END IF;


    -- 2. Mark previous assignment as not current
    UPDATE fact_assignment_history
    SET
        assignment_end_date = p_assignment_start_date - 1,
        is_current = FALSE
    WHERE employee_id = p_employee_id
      AND is_current = TRUE;


    -- 3. Insert new assignment history
    INSERT INTO fact_assignment_history (
        employee_id,
        department_id,
        job_role_id,
        assignment_start_date,
        assignment_end_date,
        transfer_reason,
        is_current
    )
    VALUES (
        p_employee_id,
        p_new_department_id,
        p_new_job_role_id,
        p_assignment_start_date,
        NULL,
        p_transfer_reason,
        TRUE
    );


    -- 4. Update employee's current department and job role
    UPDATE dim_employee
    SET
        department_id = p_new_department_id,
        job_role_id = p_new_job_role_id
    WHERE employee_id = p_employee_id;


    -- 5. Confirmation
    RAISE NOTICE
        'Employee % transferred successfully from department % / job role % to department % / job role %.',
        p_employee_id,
        v_old_department_id,
        v_old_job_role_id,
        p_new_department_id,
        p_new_job_role_id;

END;
$$;
--Test with an actual employee--
SELECT
    employee_id,
    employee_name,
    department_id,
    job_role_id
FROM dim_employee
LIMIT 10;
SELECT department_id, department_name
FROM dim_department;
SELECT job_role_id, job_title
FROM dim_job_role;
--call-
CALL sp_record_employee_transfer(
    1,
    3,
    5,
    '2024-01-01',
    'Promotion'
);
----
CREATE OR REPLACE PROCEDURE sp_record_employee_transfer(
    p_employee_id INT,
    p_new_department_id INT,
    p_new_job_role_id INT,
    p_assignment_start_date DATE,
    p_transfer_reason VARCHAR
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_old_department_id INT;
    v_old_job_role_id INT;
BEGIN

    -- 1. Get employee's current department and job role
    SELECT
        department_id,
        job_role_id
    INTO
        v_old_department_id,
        v_old_job_role_id
    FROM dim_employee
    WHERE employee_id = p_employee_id;

    -- Check employee exists
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Employee % not found', p_employee_id;
    END IF;

    -- 2. Close previous current assignment
    UPDATE fact_assignment_history
    SET
        assignment_end_date = p_assignment_start_date - 1,
        is_current = FALSE
    WHERE employee_id = p_employee_id
      AND is_current = TRUE;

    -- 3. Create new assignment
    INSERT INTO fact_assignment_history (
        employee_id,
        department_id,
        job_role_id,
        assignment_start_date,
        assignment_end_date,
        transfer_reason,
        is_current
    )
    VALUES (
        p_employee_id,
        p_new_department_id,
        p_new_job_role_id,
        p_assignment_start_date,
        NULL,
        p_transfer_reason,
        TRUE
    );

    -- 4. Update employee master
    UPDATE dim_employee
    SET
        department_id = p_new_department_id,
        job_role_id = p_new_job_role_id
    WHERE employee_id = p_employee_id;

    -- 5. Confirmation message
    RAISE NOTICE
        'Employee % transferred successfully from department % / job role % to department % / job role %.',
        p_employee_id,
        v_old_department_id,
        v_old_job_role_id,
        p_new_department_id,
        p_new_job_role_id;

END;
$$;
SELECT
    employee_id,
    employee_name,
    department_id,
    job_role_id
FROM dim_employee
LIMIT 10;
--Check departments
SELECT
    department_id,
    department_name
FROM dim_department
ORDER BY department_id;
--check jon role
SELECT
    job_role_id,
    job_title
FROM dim_job_role
ORDER BY job_role_id;
-- call 
CALL sp_record_employee_transfer(
    1,
    3,
    5,
    '2024-01-01',
    'Promotion'
);
SELECT
    employee_id,
    employee_name,
    department_id,
    job_role_id
FROM dim_employee
WHERE employee_id = 1;
--Check the assignment history

SELECT
    employee_id,
  
    ob_role_i
    assignment_start_date,
    assignment_end_date,
    transfer_reason,
    is_current
FROM fact_assignment_history
WHERE employee_id = 1
ORDER BY assignment_start_date;
