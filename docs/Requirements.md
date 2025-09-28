# CuddleCare — Software Requirements Specification (SRS)

## 1. Introduction
### 1.1 Purpose
The purpose of this document is to specify the **functional and non-functional requirements** of the **CuddleCare Childcare Booking System**.  
It serves as a reference for developers, testers, and stakeholders throughout the project lifecycle.

### 1.2 Scope
CuddleCare is a web-based platform connecting **parents/guardians** with **childcare providers**.  
- Parents can **register children, book services, track progress, and receive updates**.  
- Providers can **manage children’s records, bookings, and communication with parents**.  
- Admins oversee the system and manage users.  

### 1.3 References
- GitHub repository: [https://github.com/hanufa10/CuddleCare-ChildCare-Booking-System](https://github.com/hanufa10/CuddleCare-ChildCare-Booking-System)  
- Design mockups: (Link to Figma or screenshots)  

---

## 2. Overall Description
### 2.1 User Roles
1. **Parent/Guardian**
2. **Childcare Provider**
3. **Admin**

### 2.2 System Interfaces
- **Frontend:** React (dashboard, booking pages, communication)  
- **Backend:** PHP, MySQL (API endpoints for CRUD operations)  
- **Database:** MySQL (user data, child records, bookings, notifications)  

### 2.3 Assumptions
- Users have access to the internet and a modern web browser.  
- Providers have valid childcare licenses for verification (optional).  
- System will run locally first, then deploy to a server.  

---

## 3. Functional Requirements
### 3.1 Authentication
- Users can **register** and **login** using email and password.  
- Passwords must be securely stored (hashed).  
- Different dashboards are shown depending on user role.  

### 3.2 Parent Functions
- Add, update, and view **child profiles** (personal, health, academic info).  
- Book available **childcare services**.  
- View child **attendance, progress, and notifications**.  
- Communicate with providers via **messaging or contact form**.  

### 3.3 Provider Functions
- Manage **child profiles** for their center.  
- Record and update **attendance, health, and academic progress**.  
- View and respond to **bookings**.  
- Send updates or notifications to parents.  

### 3.4 Admin Functions
- Manage all **users and providers**.  
- Approve/reject provider registration.  
- Monitor **system usage and reports**.  
- Moderate communication if needed.  

---

## 4. Non-Functional Requirements
- **Performance:** Dashboard updates should load in < 2 seconds.  
- **Security:** Use HTTPS for secure communication.  
- **Usability:** Simple, intuitive interface for non-technical users.  
- **Availability:** System should be available 24/7 once deployed.  
- **Scalability:** System should handle multiple providers and hundreds of users.  

---

## 5. Data Requirements
- Users: Name, email, password, role  
- Children: Name, DOB, health info, academic progress  
- Bookings: Date, time, service type, status  
- Notifications: Message content, recipient, timestamp  

---

## 6. Future Enhancements
- Mobile app version (React Native or Flutter)  
- Payment integration for bookings  
- Advanced reporting and analytics for admins  

---

