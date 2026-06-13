## Day 1: Employee Onboarding Process

Today, I practiced the employee onboarding process in Active Directory using Oracle VirtualBox.

### 1. Department Setup

* Created three departmental security groups:

  * IT
  * Finance
  * Administration

### 2. Employee Onboarding

* Created a dedicated organizational unit called **Divine Employees** within the **Berlin_Office** structure.
* Added five new employee accounts.
* Assigned each employee to their respective department.

### 3. Verification

* Confirmed that all user accounts were successfully created.
* Verified that each employee was correctly assigned to the appropriate department.

### Skills Practiced

* Active Directory User Management
* Organizational Unit (OU) Administration
* Employee Onboarding
* Security Group Management
* User Account Provisioning

Screenshots of the completed configuration are attached below.


## Day 2: Identity Management & Account Security

Today, I practiced identity management in Active Directory by focusing on organizational structure, account restrictions, and account lockout policies.

### 1. Corporate Organizational Structure

* Assigned **Lina Williams** as the IT Manager.
* Linked IT employees under the IT Manager to reflect the company's reporting structure.
* Appointed **Divioline Boss** as the Chief Executive Officer (CEO).

### 2. Account Restrictions

* Configured logon hour restrictions for employees.
* Allowed IT department staff to log in between **06:00 and 18:00**.
* Allowed employees in other departments to log in between **08:00 and 18:00**.

### 3. Account Lockout Security

* Configured an account lockout threshold of **5 failed login attempts**.
* Set the account lockout duration to **5 minutes**.
* Configured the lockout counter to reset after **15 minutes** of inactivity.

### Skills Practiced

* Active Directory User Management
* Organizational Structure Design
* Logon Hour Restrictions
* Group Policy Configuration
* Account Security Management
