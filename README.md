# Project
Employee management and Attendance Tracker 

-- Department Table
CREATE TABLE Departments (
    department_id SERIAL PRIMARY KEY,
    department_name VARCHAR(100) NOT NULL
);

-- Role Table
CREATE TABLE Roles (
    role_id SERIAL PRIMARY KEY,
    role_name VARCHAR(100) NOT NULL
);

-- Employee Table
CREATE TABLE Employees (
    employee_id SERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    department_id INT REFERENCES Departments(department_id),
    role_id INT REFERENCES Roles(role_id),
    hire_date DATE,
    status VARCHAR(20) DEFAULT 'Active'
);

-- Attendance Table
CREATE TABLE Attendance (
    attendance_id SERIAL PRIMARY KEY,
    employee_id INT REFERENCES Employees(employee_id),
    attendance_date DATE NOT NULL,
    login_time TIMESTAMP,
    logout_time TIMESTAMP,
    status VARCHAR(20)
);

-- Insert into Departments
INSERT INTO Departments (department_name) VALUES
('HR'), ('Tech'), ('Finance'), ('Marketing');

-- Insert into Roles
INSERT INTO Roles (role_name) VALUES
('Manager'), ('Developer'), ('Accountant'), ('Executive');

-- Insert into Employees (extend using script, here are 10)
INSERT INTO Employees (first_name, last_name, department_id, role_id, hire_date)
VALUES
('John','Doe',1,1,'2024-09-01'),
('Jane','Smith',2,2,'2024-09-02'),
-- Add more records here...
('Emily','Clark',3,3,'2024-09-03'),
('Michael','Brown',4,4,'2024-09-04'),
('Susan','Davis',1,2,'2024-09-05'),
('Peter','Wilson',2,1,'2024-09-06'),
('Olivia','Taylor',3,4,'2024-09-07'),
('Daniel','Moore',4,3,'2024-09-08'),
('Sophia','Anderson',1,4,'2024-09-09'),
('Robert','Thomas',2,3,'2024-09-10');


-- Report: Attendance per employee, aggregated monthly
SELECT e.employee_id, e.first_name || ' ' || e.last_name AS employee_name,
       DATE_TRUNC('month', a.attendance_date) AS month,
       COUNT(*) AS days_attended
FROM Employees e
JOIN Attendance a ON e.employee_id = a.employee_id
WHERE a.status = 'Present'
GROUP BY e.employee_id, employee_name, month
ORDER BY employee_name, month;


-- Assuming late means logging in after 09:05 AM
SELECT e.employee_id, e.first_name || ' ' || e.last_name AS employee_name,
       a.attendance_date, a.login_time
FROM Employees e
JOIN Attendance a ON e.employee_id = a.employee_id
WHERE a.login_time::time > '09:05:00'
ORDER BY a.attendance_date DESC;

CREATE OR REPLACE FUNCTION set_attendance_defaults()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.login_time IS NULL THEN
        NEW.login_time := NOW();
    END IF;
    IF NEW.status IS NULL THEN
        NEW.status := 'Present';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_set_attendance_defaults
BEFORE INSERT ON Attendance
FOR EACH ROW
EXECUTE FUNCTION set_attendance_defaults();


CREATE OR REPLACE FUNCTION calc_total_work_hours(emp_id INT, month_val DATE)
RETURNS INT AS $$
DECLARE
    total_hours INT;
BEGIN
    SELECT SUM(EXTRACT(EPOCH FROM (logout_time - login_time))/3600)::INT INTO total_hours
    FROM Attendance
    WHERE employee_id = emp_id
      AND DATE_TRUNC('month', attendance_date) = DATE_TRUNC('month', month_val)
      AND status = 'Present';
    RETURN total_hours;
END;
$$ LANGUAGE plpgsql;


-- Employees with more than 20 present days in a month
SELECT employee_id, COUNT(*) AS days_present
FROM Attendance
WHERE status = 'Present'
GROUP BY employee_id, DATE_TRUNC('month', attendance_date)
HAVING COUNT(*) > 20;
