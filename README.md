# School Application Database Schema Documentation (Odoo/PostgreSQL)

## 1. Introduction

This document outlines the database schema design for a school application module within an Odoo and PostgreSQL environment. The design is based on a detailed analysis of a provided Student Admission Web Form definition and includes necessary related models to represent core school entities like students, teachers, academic standards, subjects, and accommodation.

## 2. Design Process and Methodology

The design process involved the following steps:

1.  **Analysis of the Source Data:** The provided JSON representing an Odoo Web Form (`Web Form`) was analyzed. This form defines a specific "Student Admission Application" and includes a variety of fields (`Web Form Field`) used to capture applicant and parent information, documents, academic history, health details, and more.
2.  **Identification of Core Entities:** Based on the form fields and the overall requirement for a school system, core entities were identified. The primary entity is the `Student Admission` application itself. Other necessary entities inferred were `Student` (the actual admitted person), `Teacher`, `Academic Standard` (grades/classes), `Subject`, `Academic Year`, and physical space models like `School House`, `Accommodation Area`, and `Bunk Bed`, along with transactional data like `Student Report` and `Student Attendance`. Standard Odoo base models (`res.users`, `res.country`, `res.currency`) were also included as they are linked.
3.  **Mapping Web Form Fields to Database Columns:** Each relevant field (`fieldname`) from the `web_form_fields` list in the JSON was mapped to a corresponding column name and data type in the primary `student_admission` table. Layout fields (`Section Break`, `Column Break`, `Page Break`, `HTML`) were ignored as they are for UI presentation, not data storage. Odoo field types (`Data`, `Date`, `Select`, `Link`, `Attach`, `Small Text`, `Table`, `Check`, `Phone`) were translated into appropriate PostgreSQL data types (VARCHAR, DATE, TEXT, BOOLEAN, INT/BIGINT for FKs).
4.  **Handling Child Tables:** Fields of type `Table` in the Web Form represent one-to-many relationships. For these, separate "child" tables were created (e.g., `student_admission_parent` for `parent_details`). Each child table includes a foreign key linking back to its parent `student_admission` record.
5.  **Defining Related Model Fields:** For the inferred core entities (`Student`, `Teacher`, etc.), relevant fields were added based on common requirements for managing these entities in a school database.
6.  **Establishing Relationships:** Foreign key relationships (`ref:`) were defined between tables to represent how different entities relate to each other (e.g., a Student belongs to a Standard, an Admission applies for a Standard).
7.  **Considering Odoo Specifics:** Standard Odoo fields (`id`, `name`, `create_uid`, `create_date`, `write_uid`, `write_date`) were included where applicable, recognizing Odoo's conventions. The choice of `models.Model` (Regular Model) was made as all entities represent persistent business data requiring dedicated database tables.

## 3. Odoo Model Types

All the entities designed (`student_admission`, `student`, `teacher`, etc.) are intended to be implemented as **Regular Models (`models.Model`)** in Odoo.

*   **Reasoning:** Regular Models are the standard way to define persistent data entities in Odoo. They correspond directly to database tables and are used to store business data that needs to be permanently accessible and managed within the system.
*   **Why not Transient/Abstract?:**
    *   **Transient Models (`models.TransientModel`)** are used for temporary data, typically for wizards or actions that don't need long-term storage. The entities here (applications, students, teachers) are core to the school's data and must persist.
    *   **Abstract Models (`models.AbstractModel`)** are used for inheritance purposes, providing common fields or methods to other models without having a dedicated database table themselves. While some models might inherit from base abstract models (like `mail.thread` for chatter), the core entities themselves are concrete and require tables.

## 4. Data Models and Fields

Below is a description of each data model (table) and its key fields, including reasoning for their inclusion, derived from the Web Form and inferred requirements.

### 4.1. `student_admission` (Student Application)

*   **Purpose:** Stores all the information provided by an applicant (or their parent/guardian) when submitting an admission application through the web form.
*   **Reasoning:** This is the primary record capturing the initial application data. It serves as the basis for evaluating the applicant and, if successful, creating a `student` record.
*   **Key Fields & Reasoning:**
    *   `name`: Standard Odoo display name, likely a sequence number (e.g., ADM/YYYY/NNNN) for easy identification.
    *   `applicant_user_id`: Link to the `res_users` record of the user who submitted the application. *Reasoning: Identifies the submitting user for tracking and access control.*
    *   `student_id`: Link to the `student` record if the applicant is admitted. *Reasoning: Connects the application to the eventual student record for tracing.* (Nullable as not all applications result in admission).
    *   `application_year_id`: Link to `academic_year`. *Reasoning: Specifies the academic year for which the application is made.* (Required as per form).
    *   `applied_for_grade_id`: Link to `academic_standard`. *Reasoning: Specifies the grade/standard the applicant is applying to enter.* (Required as per form).
    *   `applied_before`: Select (Yes/No). *Reasoning: Tracks if the applicant has applied previously.*
    *   `prev_application_year_id`, `prev_applied_for_grade_id`, `prev_application_remarks`: Conditional fields to capture details of previous applications. *Reasoning: Useful context for evaluation.*
    *   `applicant_full_name`, `date_of_birth`, `gender`, `nationality_id`, `country_of_birth_id`, `country_of_residence_id`: Core personal identification details. *Reasoning: Essential for applicant identity verification and eligibility.* (Many required).
    *   `other_gender_details`: Conditional text field. *Reasoning: Allows specifying gender if 'Other' is selected.*
    *   `comm_address_...`: Fields for communication address details. *Reasoning: Needed for correspondence.* (Many required). Linked to `res_country`.
    *   `identification_mark1`, `identification_mark2`: Physical identification marks. *Reasoning: For physical identification.* (Required).
    *   `religion`, `community`, `other_..._details`: Demographic information. *Reasoning: May be required for school reporting or internal demographics.* (Required, with conditional 'Other' details).
    *   `mother_tongue`, `second_language_id`, `third_language_id`: Language details. *Reasoning: Academic placement and understanding language background.* (`mother_tongue` required, others conditional links to `subject` or could be text).
    *   `has_sibling_in_ihs`: Select (Yes/No). *Reasoning: Sibling presence can be a factor in admissions.* (Required).
    *   `recent_photograph_path`, `birth_certificate_path`, `id_proof_document_path`: Paths to uploaded document files. *Reasoning: Stores uploaded documents required for application verification.* (Required).
    *   `id_proof_type`, `aadhaar_number`, `passport_number`, `passport_place_of_issue`, `passport_date_of_issue`, `passport_date_of_expiry`: Details about the provided ID proof. *Reasoning: Verification of identity and document validity.* (`id_proof_type` required, others conditional).
    *   `is_currently_home_schooled`, `was_ever_home_schooled`: Schooling history type. *Reasoning: Understand the applicant's educational background.* (`is_currently_home_schooled` required).
    *   `current_school_...`, `previous_schools_...`: Fields/links related to current and previous schools. *Reasoning: Academic history and background check.* (Conditional based on `is_currently_home_schooled` and `studied_in_school_previously`). `board_affiliation_data2` was mapped as `current_school_board_affiliation` (Data), while `board_affiliation` was mapped as `current_school_board_affiliation_id` (Link to `school_board_affiliation`), potentially one is redundant or used differently.
    *   `emis_id`: Conditional field for Indian state-specific ID. *Reasoning: Required statutory data in a specific region.*
    *   `academic_strengths_weaknesses`, `hobbies_interests_activities`, `temperament_personality`, `learning_needs_disability`, `other_important_details`: Open-text responses from the applicant/parent. *Reasoning: Provides qualitative insight into the applicant.* (Many required).
    *   `protected_..._vaccine`: Select (Yes/No) fields for various vaccinations. *Reasoning: Basic health screening requirement.* (Required).
    *   `other_vaccinations_details`, `vaccine_certificates_path`: Details/documents for other vaccines. *Reasoning: Comprehensive health record.* (`vaccine_certificates_path` required).
    *   `blood_group`: Select. *Reasoning: Essential medical information.* (Required).
    *   `wears_glasses_lenses`, `left_eye_power`, `right_eye_power`: Vision correction details. *Reasoning: Important health detail.* (`wears_glasses_lenses` required, power conditional).
    *   `is_toilet_trained`, `wets_bed`: Hygiene training details (Conditional for younger grades). *Reasoning: Relevant for readiness assessment in specific age groups.*
    *   `has_hearing_challenges`, `hearing_challenges_details`, etc. (for physical, behavioural, speech challenges): Details on various health/developmental challenges. *Reasoning: Understand potential needs and required support.* (Required selects, conditional text details).
    *   `has_history_injury`, `history_injury_details`, etc. (for health issue, hospitalization): History of medical events. *Reasoning: Assess past medical history.* (Required selects, conditional text details).
    *   `is_on_regular_medication`, `regular_medication_details`, `medical_prescription_path`: Current medication information. *Reasoning: Critical for ongoing medical care.* (`is_on_regular_medication` required, others conditional).
    *   `has_allergies`, `allergy_details`: Allergy information. *Reasoning: Essential safety information.* (`has_allergies` required, details conditional).
    *   `parent_marital_status`: Select. *Reasoning: Impacts legal rights and contact procedures.* (Required).
    *   `tuition_payer`, `court_order_document_path`, `visitation_rights_holder`, `school_communication_recipient`, `report_card_recipient`, `legal_rights_document_path`: Conditional fields related to legal/financial rights for divorced parents. *Reasoning: Addresses specific legal arrangements.* (Conditional).
    *   `parents_are_local_guardians`: Select (Yes/No). *Reasoning: Determines if additional guardians need to be recorded.*
    *   `subject_group_a` to `subject_group_d`: Selected subject options (Conditional for Class XI). *Reasoning: Records subject preferences for higher grades.*
    *   `q1_applicant_response` to `q7_applicant_response`, `q1_parent_response` to `q6_parent_response`: Responses to essay/question prompts. *Reasoning: Qualitative assessment as part of the application process.* (Conditional).
    *   `billing_full_name`, `billing_email`, `billing_phone`, `billing_address_...`: Details for billing the application fee. *Reasoning: Required for payment processing.* (Many required). Linked to `res_country`.
    *   `agreed_to_terms`: Checkbox. *Reasoning: Legal declaration of agreement.* (Required).
    *   `submission_date`, `submission_place`: Date and location of application submission. *Reasoning: Recordkeeping.* (Required).
    *   `application_fee_status`: Select. *Reasoning: Tracks the payment status of the application fee.* (Likely system-updated).
    *   `program_id`: Link to `program`. *Reasoning: Identifies the specific fee program linked to this application.* (Required).
    *   `amended_from_application_id`: Link to another `student_admission` record. *Reasoning: Tracks application amendments/revisions.* (Readonly).
    *   `internal_..._feedback_status`, `internal_..._feedback_notes`: Fields for internal school staff evaluation feedback. *Reasoning: Records assessment outcomes and notes within the application record.*

### 4.2. Child Tables of `student_admission`

*   **Purpose:** Store lists of related information linked to a single `student_admission` record.
*   **Reasoning:** Used for information where there can be multiple entries for one application (e.g., multiple parents, multiple siblings). Odoo's 'Table' field type maps directly to these one-to-many structures. Each table includes an `admission_id` FK linking back to the parent `student_admission`.

    *   `student_admission_language_proficiency`: Stores details on *other* languages and proficiency levels beyond the main ones.
    *   `student_admission_sibling`: Stores details for each sibling the applicant has.
    *   `student_admission_previous_school`: Stores details for each previous school the applicant attended.
    *   `student_admission_parent`: Stores detailed information for each parent or primary guardian figure.
    *   `student_admission_guardian`: Stores detailed information for other designated guardians.
    *   `student_admission_payment_link`: Stores references to specific payment transactions generated for this application.

### 4.3. `academic_standard` (Grades/Classes)

*   **Purpose:** Master data defining the different school grades or classes.
*   **Reasoning:** Provides a structured list for assigning students, teachers, and curriculum.

### 4.4. `subject` (Academic Subjects)

*   **Purpose:** Master data defining the subjects taught.
*   **Reasoning:** Provides a structured list for curriculum definition, linking subjects to standards, and potentially recording student performance in reports.

### 4.5. `academic_year` (School Years)

*   **Purpose:** Master data defining the school academic years.
*   **Reasoning:** Provides a structured list for time-sensitive data like applications, enrollments, and reports.

### 4.6. `student` (Admitted Student)

*   **Purpose:** The core entity representing a person currently enrolled in the school.
*   **Reasoning:** Stores ongoing information about a student after they have been admitted. Much of the initial data comes from the `student_admission` record.

### 4.7. `teacher` (School Staff)

*   **Purpose:** Represents staff members, particularly those involved in teaching, guidance, or house management.
*   **Reasoning:** Manages personnel records, links to Odoo users, and allows assigning roles (class teacher, warden).

### 4.8. `school_house` (Residential Houses)

*   **Purpose:** Represents the residential divisions of the school (e.g., boarding houses).
*   **Reasoning:** Manages accommodation structure and assigns students and wardens.

### 4.9. `accommodation_area` (Areas within Houses)

*   **Purpose:** Represents sub-divisions within a school house (e.g., specific dorm rooms, floors).
*   **Reasoning:** Breaks down houses into manageable units for assigning beds.

### 4.10. `bunk_bed` (Individual Beds)

*   **Purpose:** Represents the smallest unit of accommodation.
*   **Reasoning:** Tracks individual bed spaces and their occupancy by students.

### 4.11. `student_report` (Academic Reports)

*   **Purpose:** Stores records of student academic performance reports.
*   **Reasoning:** Provides historical and current academic progress information.

### 4.12. `student_attendance` (Daily Attendance)

*   **Purpose:** Records the daily attendance status for each student.
*   **Reasoning:** Tracks student presence/absence over time.

### 4.13. `school_board_affiliation` (Board Master Data)

*   **Purpose:** Master data for external academic board affiliations.
*   **Reasoning:** Provides a lookup list for school board names.

### 4.14. `program` (Program/Fee Master Data)

*   **Purpose:** Master data defining various programs, including fee structures like the application fee.
*   **Reasoning:** Provides a lookup list for programs associated with applications and payments.

### 4.15. Standard Odoo Models (`res_users`, `res_country`, `res_currency`)

*   **Purpose:** Represent essential built-in Odoo data like users, countries, and currencies.
*   **Reasoning:** It is standard Odoo practice to link to these existing master data models rather than duplicating their information. `res_users` is also linked as the creator/modifier of records.

## 5. Relationships Explained

Relationships connect the tables, enabling data integrity and allowing queries across related information.

*   **`student_admission` links to Master Data:**
    *   An Admission application is for one `academic_year` and one `academic_standard`. (`student_admission` `> academic_year`, `> academic_standard`).
    *   An Admission application is linked to one `res_users` who submitted it. (`student_admission` `> res_users`).
    *   An Admission links to various `res_country` records (nationality, birth, residence, addresses). (`student_admission` `> res_country`).
    *   An Admission links to a `school_board_affiliation` for the current school details. (`student_admission` `> school_board_affiliation`).
    *   An Admission links to a `program` to define the application fee or program type. (`student_admission` `> program`).

*   **`student_admission` One-to-Many with Child Tables:**
    *   One `student_admission` can have many `student_admission_language_proficiency` records. (`student_admission` `< student_admission_language_proficiency`).
    *   One `student_admission` can have many `student_admission_sibling` records. (`student_admission` `< student_admission_sibling`).
    *   One `student_admission` can have many `student_admission_previous_school` records. (`student_admission` `< student_admission_previous_school`).
    *   One `student_admission` can have many `student_admission_parent` records. (`student_admission` `< student_admission_parent`).
    *   One `student_admission` can have many `student_admission_guardian` records. (`student_admission` `< student_admission_guardian`).
    *   One `student_admission` can have many `student_admission_payment_link` records. (`student_admission` `< student_admission_payment_link`).

*   **`student` links to Academic & Accommodation Structure:**
    *   A `student` is enrolled in one `academic_year` and one `academic_standard`. (`student` `> academic_year`, `> academic_standard`).
    *   A `student` resides in one `school_house`, one `accommodation_area`, and occupies one `bunk_bed`. (`student` `> school_house`, `> accommodation_area`, `> bunk_bed`). Note: The `bunk_bed` also links back to the student who occupies it (`bunk_bed` `> student`).
    *   A `student` is linked to their original `student_admission` application. (`student` `> student_admission`).

*   **Academic Structure Relationships:**
    *   An `academic_standard` can be headed by one `teacher`. (`academic_standard` `> teacher`).
    *   `subject` and `academic_standard` have a many-to-many relationship, managed by the `subject_academic_standard_rel` junction table. (`academic_standard` `||--o{ subject_academic_standard_rel`, `subject` `||--o{ subject_academic_standard_rel`).
    *   `academic_standard` is used to define the grade for `student_report` and `student_attendance` records. (`academic_standard` `< student_report`, `< student_attendance`).
    *   `subject` is optionally linked in `student_admission` for language choices. (`subject` `< student_admission`).

*   **Personnel & Accommodation Relationships:**
    *   A `teacher` is linked to one `res_users` account. (`teacher` `> res_users`).
    *   A `teacher` can be assigned as the warden of one `school_house`. (`teacher` `> school_house`).
    *   A `school_house` contains many `accommodation_area`s. (`school_house` `||--o{ accommodation_area`).
    *   An `accommodation_area` contains many `bunk_bed`s. (`accommodation_area` `||--o{ bunk_bed`).
    *   A `teacher` can record many `student_attendance` entries. (`teacher` `< student_attendance`).

*   **Transactional Data Relationships:**
    *   A `student_report` is created for one `student`, in one `academic_year`, for one `academic_standard`. (`student_report` `> student`, `> academic_year`, `> academic_standard`).
    *   A `student_attendance` record is for one `student`, on a specific date, in a specific `academic_standard`, and recorded by one `teacher`. (`student_attendance` `> student`, `> academic_standard`, `> teacher`).
    *   A `student_admission_payment_link` is generated for one `program`. (`student_admission_payment_link` `> program`).

*   **Standard Odoo Relationships:**
    *   Almost all tables have `create_uid` and `write_uid` linking to `res_users`. (`...` `> res_users`).
    *   `program` links to `res_currency` for the fee amount. (`program` `> res_currency`).

## 6. Potential Simplifications and Improvements

While the current design directly reflects the Web Form structure and covers core school needs, here are some areas that could be simplified or improved, particularly in an Odoo context:

1.  **Leverage `res.partner`:** Odoo has a robust `res.partner` model for managing contacts (individuals and companies). Instead of creating separate fields or child tables for `student_admission_parent` and `student_admission_guardian`, consider creating `res.partner` records for each parent/guardian and linking them to the `student_admission` (and later, the `student`) record. This centralizes contact information and leverages built-in Odoo features (addresses, communication history, etc.).
2.  **Consolidate Address Fields:** The `student_admission` table has separate fields for communication address and billing address. If these are often the same, UI logic can pre-fill them. However, for simplification in the database, one could potentially use Odoo's multi-address features on a linked `res.partner` or consolidate addresses slightly if the form structure allows (though the current explicit fields directly map to the form).
3.  **Simplify `current_school_board_affiliation`:** The Web Form has both a Data field and a Link field for board affiliation. Decide on one primary way to store this (the Link to `school_board_affiliation` is preferred for structured data) and potentially remove the redundant Data field from the model after the data is migrated or if the form is adjusted.
4.  **Nationality/Country Fields:** While reflecting the form, having separate links for nationality, birth country, residence country on `student_admission` is detailed but potentially redundant with a linked `res.partner` approach for the applicant themselves (if the student is also a partner). The current approach is clearer for mapping the form directly.
5.  **Dynamic Subject Selection (Class XI):** Instead of storing selected options as strings or separate fields on the `student_admission` record, consider creating a child table on the `student` model (or `student_academic_year` relation) specifically for `student_subjects`. This would allow storing the actual subjects taken by the student *after* admission, linked to the `subject` master data, offering more flexibility for tracking student academic paths over time. The subject preferences on the `student_admission` could remain as initial choices.
6.  **Evaluation Fields Placement:** The internal evaluation fields (`internal_...`) are currently on the `student_admission` record. If the evaluation process is complex or involves multiple evaluators, consider a separate `student_admission_evaluation` child table or a dedicated evaluation model to store multiple rounds or types of feedback.
7.  **Computed Fields:** Fields like `applicant_age` are typically computed in Odoo based on the `date_of_birth` and the current date/application date, rather than stored as a physical column.

These potential improvements focus on leveraging Odoo's existing data structures (`res.partner`), reducing potential data redundancy, and designing for flexibility beyond the initial application phase. The current DBML design is a solid foundation directly mirroring the required data capture.
