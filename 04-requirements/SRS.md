# SOFTWARE REQUIREMENTS SPECIFICATION

**PROJECT: SOFTWARE FOR THE CREATION OF THE "ELECTRICAL CONSUMPTION MONITOR" APPLICATION**

**MEMBERS:**

**MIGUEL ANGEL GARCIA ARTUNDUAGA**

**JUAN ESTEBAN CORTES**

**INSTRUCTOR:**

**MOTTA VARGAS JOSÉ DE JESÚS**

**SERVICIO NACIONAL DE APRENDIZAJE -- SENA SOFTWARE PROGRAMMING -- 3145556**

#### 2025

## Table of Contents

1. Introduction
   1. Problem Statement
   2. Purpose
   3. Justification
   4. General Objective
      1. Specific Objectives
   5. Scope
   6. Personnel Involved
   7. Definitions, Acronyms, and Abbreviations
   8. References
   9. Summary
2. General Description
   1. Product Perspective
   2. User Characteristics
   3. Constraints
   4. Assumptions and Dependencies
3. Specific Requirements
   1. Common Interface Requirements
      1. User Interfaces
      2. Hardware Interfaces
      3. Software Interfaces
      4. Communication Interfaces
   2. Functional Requirements
   3. Non-Functional Requirements
4. Use Case Requirements
   1. UML Use Case Diagram
   2. Use Case Characterization

# 1. Introduction

Welcome to "Electrical Consumption Monitor," a window into a more conscious, efficient, and sustainable life in Colombian households. In a country where access to energy is available to everyone, we are often not fully aware of how we use it, how much we waste, and the impact this has on the environment and our economy. Our application is not just a control tool — it is an invitation to transform habits, save money, and protect the planet from the comfort of home.

"Electrical Consumption Monitor" was born as a response to an urgent need: the lack of clear, accessible information about how, when, and how much we consume. This project seeks to empower every user with real, real-time data that lets them understand which devices consume the most energy, when consumption peaks occur, and what actions they can take to reduce them.

From an unnecessarily lit lightbulb to appliances that consume excessive energy, every detail matters. With a friendly, functional interface, users will be able to view their consumption hour by hour, receive alerts when levels exceed set limits, and access personalized recommendations to optimize their energy use.

Whether you are looking to reduce your monthly bill or actively contribute to caring for the environment, "Electrical Consumption Monitor" adapts to your needs. This platform accompanies you on the path toward more responsible habits, offering clear tools to make informed and sustainable decisions.

It also fosters a community that shares good practices, experiences, and energy awareness. "Electrical Consumption Monitor" is more than an application: it is a movement that promotes more efficient homes, more informed citizens, and a country committed to a cleaner, more balanced energy future.

## 1.1 Problem Statement

The high energy consumption in Colombian households is largely due to a lack of energy awareness. Many people are not aware of the environmental impact of leaving a lightbulb on unnecessarily or letting an appliance, such as a refrigerator, consume more energy than required. This lack of control and knowledge about daily energy consumption contributes to environmental deterioration and rising costs of energy services across different households.

## 1.2 Purpose

This document aims to define the use cases through their characterization and representation by means of UML diagrams, as well as to establish the functional and non-functional requirements for the development of an application that allows users to monitor energy consumption in their homes in real time. This tool seeks to promote the efficient use of electrical energy, offering a clear view of consumption, alerts for excesses, and personalized recommendations. In addition, the application must have an intuitive interface and be secure and reliable for both regular users and system administrators, providing an accessible, useful, and sustainability-focused experience.

## 1.3 Justification

Currently, most households do not have tools that allow them to know their real-time energy consumption or identify which electrical appliances consume the most. This situation leads to inefficient energy use, generating high costs on electricity bills and a negative impact on the environment. Implementing a technological solution that allows monitoring energy consumption would help foster responsible and sustainable electricity usage habits.

## 1.4 General Objective

Develop an energy resource control system that allows real-time monitoring of energy consumption in households. This tool will facilitate the identification of appliances that generate the highest expense, thereby promoting more efficient and responsible use of electricity.

### 1.4.1 Specific Objectives

- Design a system that measures real-time energy consumption for each device or appliance in the home.
- Develop a mobile application or digital platform that clearly and accessibly displays energy consumption information.
- Implement automatic alerts or notifications that warn about unusual or excessive consumption in the monitored devices.
- Enable the generation of periodic reports on energy usage, helping users make decisions to reduce their consumption.
- Foster responsible and sustainable habits in the use of electrical energy through personalized recommendations based on the data obtained by the system.

## 1.5 Scope

This SRS specifies the software requirements of the system for creating an application:

1.5.1. An application that allows users to record consumption thresholds or set default thresholds and manually monitor their electricity consumption.

1.5.2. Configuration of custom consumption thresholds and notification frequency, to alert the user when approaching or exceeding their set limits.

1.5.3. Generation of detailed analyses of electrical consumption behavior, including personalized recommendations for improving energy efficiency and reducing costs.

## 1.6 Personnel Involved

| Field                     | Details                                                    |
| ------------------------- | ---------------------------------------------------------- |
| **Name**                  | Juan Esteban Cortes Parra                                  |
| **Role**                  | Project Lead, Database Manager, Developer                  |
| **Professional Category** | Trainee, Technologist in Software Analysis and Development |
| **Responsibility**        | Information analysis, design, and programming              |
| **Contact Information**   | juanestebanp171@gmail.com                                  |

| Field                     | Details                                                    |
| ------------------------- | ---------------------------------------------------------- |
| **Name**                  | Miguel Ángel García Artunduaga                             |
| **Role**                  | Analyst, Developer                                         |
| **Professional Category** | Trainee, Technologist in Software Analysis and Development |
| **Responsibility**        | Information analysis, design, and programming              |
| **Contact Information**   | miguel.garcia1111111@soy.sena.edu.co                       |

## 1.7 Definitions, Acronyms, and Abbreviations

| **_Term_** | **_Description_**                                                        |
| ---------- | ------------------------------------------------------------------------ |
| **User**   | Person who will use the application.                                     |
| **SRS**    | Software Requirements Specification.                                     |
| **FR**     | Functional Requirement.                                                  |
| **NFR**    | Non-Functional Requirement.                                              |
| **SENA**   | Servicio Nacional de Aprendizaje (Colombia's National Training Service). |
| **UC**     | Use Case.                                                                |

## 1.8 References

| **Document Title**       | **Reference** |
| ------------------------ | ------------- |
| Standard IEEE 830 - 1998 | IEEE          |
| APA 7 Standards          |               |

## 1.9 Summary

The application is divided into three main sections:

**Registration and login:** In this section, the user can create an account or access their existing profile.

**Threshold configuration:** The user can set energy consumption limits and define how often they wish to receive usage notifications.

**Analysis and recommendations:** The application will provide a detailed analysis of the user's energy consumption, along with personalized suggestions to improve efficiency and reduce expenses.

# 2. General Description

## 2.1 Product Perspective

The application aims to provide users with accurate and up-to-date information about their energy consumption, facilitating efficient control of it. In addition, the system generates automatic alerts when high consumption is detected, allowing the user to make timely decisions to regulate their electricity spending.

## 2.2 User Characteristics

| Field          | Details                                                                                                                                                                                                                                    |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Username**   | Administrator                                                                                                                                                                                                                              |
| **Background** | Technologist in Software Analysis and Development                                                                                                                                                                                          |
| **Activities** | Manages and oversees all software application functionalities. Responsible for system administration, error handling and resolution, as well as preventive and corrective maintenance of the application to ensure its proper functioning. |

| Field          | Details                                                                                   |
| -------------- | ----------------------------------------------------------------------------------------- |
| **Username**   | Home Owner                                                                                |
| **Background** | Household administrator                                                                   |
| **Activities** | Person responsible for managing a household, including managing linked devices and users. |

| Field          | Details                                                                                                                                                                                                                                                      |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Username**   | Home Member                                                                                                                                                                                                                                                  |
| **Background** | Natural (individual) user                                                                                                                                                                                                                                    |
| **Activities** | Interacts with the application to view information related to their energy consumption. Can manually record consumption data, configure spending thresholds and notification frequency, and access reports and recommendations to optimize their energy use. |

## 2.3 Constraints

**2.3.1.** The application requires internet access to sync data, send notifications, and generate real-time analyses.

**2.3.2.** Users must have the latest version of the application installed to ensure system compatibility and security.

**2.3.3.** The software must be compatible with Android mobile devices, as well as updated web browsers (Chrome, Firefox, Safari, Edge).

**2.3.4.** Manual recording of consumption thresholds depends on how much energy the user needs to save.

**2.3.5.** Consumption optimization suggestions are based on estimated patterns and do not replace specialized technical advice.

**2.3.6.** The notification system may be affected by custom device settings or battery-saving restrictions.

## 2.4 Assumptions and Dependencies

1. It is assumed that the requirements described here are stable.
2. The devices on which the application will run must meet the stated requirements to ensure its correct execution.
3. The design is created and unified using data provided by SENA trainees and instructors.
4. It is assumed that the data and information provided to the user are updated in real time.
5. The software may depend on user participation to provide accurate and up-to-date information.
6. A constant capacity to update and add content, such as new features, is assumed in order to keep the app easy to use.

# 3. Specific Requirements

## 3.1 Common Interface Requirements

### 3.1.1 User Interfaces

The user interface must be accessible, intuitive, responsive, and consistent, with fast loading times, clear navigation, immediate feedback, and compatibility across web and mobile platforms, ensuring a smooth and usable experience for all users.

### 3.1.2 Hardware Interfaces

Computers and mobile devices in good working condition will be required, with the following characteristics:

- Internet connection.
- The most recent version of the software and operating system.
- Sufficient storage to save the application.

### 3.1.3 Software Interfaces

- Operating System: Windows 7 or higher for computers.
- Operating System: the latest available version of Android.
- Browser: Mozilla, Chrome, Edge, Brave, Vivaldi, among others.

### 3.1.4 Communication Interfaces

Servers and applications will communicate with each other using standard internet protocols whenever possible. For example, to transfer files or documents, existing protocols (FTP or other suitable ones) must be used.

## 3.2 Functional Requirements

**FR:** Functional Requirement

**FRS:** Functional Requirement Specification

### FR1. User Management

**FRS1.1:** Registration and authentication.

- The system must allow user registration and authentication, both on the web version and the mobile application.

**FRS1.1.1:** Authentication and registration with social networks.

- The user may log in or register using external authentication providers (social networks), which will be stored as part of their user configuration.

**FRS1.2:** Log in.

**FRS1.3:** User profile.

**FRS1.3.1:** Update information.

- The system must allow users to edit and update their personal information.

**FRS1.3.2:** Recover password.

- The system must allow users to reset their password if forgotten, via email or another secure verification method.
- This ensures continued access and prevents permanent account lockouts.

**FRS1.3.3:** Delete account.

- Users may delete their account if they no longer wish to use the service.

#### FR1 Specification Sheet

| Field                                    | Details                                                                                                                                                                                                                                                                                                                                                                              |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Identifier**                           | RF1                                                                                                                                                                                                                                                                                                                                                                                  |
| **Name**                                 | User Management                                                                                                                                                                                                                                                                                                                                                                      |
| **Type**                                 | Functional                                                                                                                                                                                                                                                                                                                                                                           |
| **Requirement that uses/specializes it** | Used by RF4                                                                                                                                                                                                                                                                                                                                                                          |
| **Critical?**                            | Yes                                                                                                                                                                                                                                                                                                                                                                                  |
| **Development priority**                 | High                                                                                                                                                                                                                                                                                                                                                                                 |
| **Associated visualization documents**   | Registration screen, Login screen, Profile screen, Password recovery screen                                                                                                                                                                                                                                                                                                          |
| **Input**                                | User data (first name, last name, email, password); login credentials; updated profile-edit data; acceptance of terms and permissions; email for password recovery                                                                                                                                                                                                                   |
| **Output**                               | Account created; session started; profile updated; recovery link or code sent to the user                                                                                                                                                                                                                                                                                            |
| **Description**                          | The system must manage user registration, authentication, and administration, allowing access from both the web version and the mobile application. It must include functionality for profile updates, password recovery via secure methods, and social-network authentication. In addition, the user must accept the terms, conditions, and permissions required to use the system. |
| **Abnormal situation handling**          | Email already registered: the user is notified. Incorrect password: access is denied. Failed password recovery: resend code or report an error. Third-party authentication failure: show a message and allow retry.                                                                                                                                                                  |

### FR2. Project Management

**FRS2.1:** Register project or home.

- The system must allow the responsible user to register a project (e.g., a house, building, or company).

**FRS2.2:** User permission management.

- The system must assign permissions to users according to their role (regular user or responsible user), enabling information handling and determining the actions they can perform within the system, such as managing devices and projects.

**FRS2.3:** Device linking.

- The responsible user registers and links devices to the project, and links users (regular users) to the project.

**FRS2.4:** Responsible user.

- Each project or home must have a responsible user in charge of managing all devices. This role has advanced platform permissions.

**FRS2.5:** Management of favorite projects or homes.

- The system must allow any user to mark projects or homes to which they have access as favorites, in order to make access to them easier and faster.

#### FR2 Specification Sheet

| Field                                    | Details                                                                                                                                                                                                                                                                                                                                                              |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Identifier**                           | RF2                                                                                                                                                                                                                                                                                                                                                                  |
| **Name**                                 | Project and Home Management                                                                                                                                                                                                                                                                                                                                          |
| **Type**                                 | Functional                                                                                                                                                                                                                                                                                                                                                           |
| **Requirement that uses/specializes it** | Uses RF3, RF4                                                                                                                                                                                                                                                                                                                                                        |
| **Critical?**                            | Yes                                                                                                                                                                                                                                                                                                                                                                  |
| **Development priority**                 | High                                                                                                                                                                                                                                                                                                                                                                 |
| **Associated visualization documents**   | Screen for registering a project/home; interface for favorite projects or homes; form for linking devices; administration panel for the responsible user                                                                                                                                                                                                             |
| **Input**                                | Project data (home/company name, location, description); devices to link; list of linked users and assigned responsible user; list of frequent homes                                                                                                                                                                                                                 |
| **Output**                               | Project registered; devices linked to the home; responsible user assigned; project listed                                                                                                                                                                                                                                                                            |
| **Description**                          | The system allows registering projects or homes and linking users and devices to those environments. Each project must have a responsible user, who has advanced permissions to manage devices, secondary users, and key settings. This functionality is essential to ensure agile, structured, and organized management of energy consumption per home or facility. |
| **Abnormal situation handling**          | Unauthorized user attempts to register or modify projects: deny the action. Device already linked to another project: notify of the conflict. Project without a responsible user: block until one is assigned. Accidental removal of the responsible user: allow immediate reassignment.                                                                             |

### FR3. System Management and Configuration

**FRS3.1:** Threshold configuration.

- The user will be able to configure consumption thresholds in order to receive personalized alerts and recommendations. In addition, the system will include preset default thresholds according to the user's needs.

**FRS3.2:** Color configuration.

- The user will be able to customize the system's colors. There will be predefined color palettes that the user can select as desired. These will automatically adapt to the system's dark mode.

**FRS3.3:** Language configuration.

- The user will be able to configure the system language according to their preferences. Up to 4 different languages will be available.

**FRS3.4:** Alert configuration.

- The user will be able to configure how they want to receive alerts, whether through notifications or through some other digital medium such as email.

**FRS3.5:** Global system configuration.

- The system must have a predefined global configuration that establishes default values such as consumption limits, language, and notifications, which can be used as the basis for each user's individual configuration.

#### FR3 Specification Sheet

| Field                                    | Details                                                                                                                                                                                                                                                                                                                                                                                                           |
| ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Identifier**                           | RF3                                                                                                                                                                                                                                                                                                                                                                                                               |
| **Name**                                 | System Management and Configuration                                                                                                                                                                                                                                                                                                                                                                               |
| **Type**                                 | Functional                                                                                                                                                                                                                                                                                                                                                                                                        |
| **Requirement that uses/specializes it** | Uses RF4                                                                                                                                                                                                                                                                                                                                                                                                          |
| **Critical?**                            | Yes                                                                                                                                                                                                                                                                                                                                                                                                               |
| **Development priority**                 | Medium-High                                                                                                                                                                                                                                                                                                                                                                                                       |
| **Associated visualization documents**   | Global system configuration screen; language selection; theme/color selection; alert-sending selection; threshold configuration panel                                                                                                                                                                                                                                                                             |
| **Input**                                | Custom thresholds; color palette selection; system language; notification preferences (app, email, both); adjustment of default values                                                                                                                                                                                                                                                                            |
| **Output**                               | Saved settings; system language change; visual adjustment applied to the system; alerts configured according to preferences; updated threshold configuration panel                                                                                                                                                                                                                                                |
| **Description**                          | This module allows the user to customize the general operation of the system, including electrical consumption thresholds, color palettes adaptable to dark mode, preferred language, and notification methods. Its purpose is to provide a flexible, accessible experience tailored to the individual needs of each user or project. These settings directly influence alert generation and information display. |
| **Abnormal situation handling**          | Selected language not available: keep the current language. Invalid threshold values: reject and request correction. Error saving configuration: allow retry and notify. Incompatible color palette: adjust automatically.                                                                                                                                                                                        |

### FR4. Recording and Processing of Electrical Consumption

**FRS4.1:** Automatic recording of consumption.

- The system must automatically record the electrical consumption coming from connected devices in a manner that can be parameterized by the user.

**FRS4.2:** Consumption visualization.

**FRS4.2.1:** Selection of consumption date ranges.

- The user must be able to select date ranges to analyze consumption patterns.

**FRS4.2.2:** Consumption comparison.

- The system must allow comparing current consumption with historical averages.

**FRS4.2.3:** Consumption trend charts.

- The system must offer a panel with consumption trend charts (peaks, critical time periods, and monthly comparisons).

**FRS4.3:** Generation of automatic alerts.

- The system must automatically generate alerts based on the analysis of received measurements, using internal mechanisms that evaluate consumption levels in real time.

**FRS4.3.1:** Alert visualization.

- The user will be able to view the alerts generated by the system and resolve them depending on the notification given.

**FRS4.4:** Generation of automatic recommendations.

- The system must generate automatic recommendations based on the electrical consumption behavior detected in the measurements, even when no critical situations occur.

**FRS4.4.1:** Recommendation visualization.

- The user will be able to view the recommendations generated by the system and mark them as read.

#### FR4 Specification Sheet

| Field                                    | Details                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| ---------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Identifier**                           | RF4                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| **Name**                                 | Recording and Processing of Electrical Consumption                                                                                                                                                                                                                                                                                                                                                                                            |
| **Type**                                 | Functional                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| **Requirement that uses/specializes it** | Uses RF1, RF2, RF3, and RF5                                                                                                                                                                                                                                                                                                                                                                                                                   |
| **Critical?**                            | Yes                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| **Development priority**                 | Very high                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| **Associated visualization documents**   | Trends panel; comparative charts; consumption tables; view of generated alerts and recommendations                                                                                                                                                                                                                                                                                                                                            |
| **Input**                                | Hourly consumption data sent by devices; configured thresholds; selected date range; request for comparison or historical analysis; report format                                                                                                                                                                                                                                                                                             |
| **Output**                               | Records stored in the database; automatic alerts generated; trend charts; consumption comparisons; report generator (PDF)                                                                                                                                                                                                                                                                                                                     |
| **Description**                          | The system is responsible for automatically receiving and storing hourly energy consumption data from linked devices. Based on this information, it generates trends, allows filtering by date ranges, performs historical comparisons, and detects excessive consumption through automatic alerts. This functionality is the operational core of the system, providing the user with key information for decision-making and energy savings. |
| **Abnormal situation handling**          | Data reception failure: retry and notify. Corrupted data: discard and log the error. Poorly configured threshold: use default thresholds. Disconnected device: generate a special alert.                                                                                                                                                                                                                                                      |

### FR5. Measurement Device Management

**FRS5.1:** Data transmission to the system.

- The measurement device must automatically send electrical consumption data (voltage, current, active power, and accumulated energy) to the system at regular 60-second intervals via the MQTT protocol, so that the Broker can transmit consumption information to subscribed clients.

**FRS5.2:** Linking the device with the system.

- The system must recognize and authenticate devices using a unique identifier and an access key, allowing their secure linking with a project.

**FRS5.3:** Device connection status.

- The system must monitor the device's status (online / offline) based on the last measurement received.

**FRS5.4:** Automatic reconnection.

- The device must be capable of automatically reconnecting to the system in the event of WiFi connection loss or a temporary communication failure, without user intervention.

#### FR5 Specification Sheet

| Field                                    | Details                                                                                                                                                                                                                                                                                      |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Identifier**                           | RF5                                                                                                                                                                                                                                                                                          |
| **Name**                                 | Measurement Device Management                                                                                                                                                                                                                                                                |
| **Type**                                 | Functional                                                                                                                                                                                                                                                                                   |
| **Requirement that uses/specializes it** | Used by RF2, RF4                                                                                                                                                                                                                                                                             |
| **Critical?**                            | Yes                                                                                                                                                                                                                                                                                          |
| **Development priority**                 | Very high                                                                                                                                                                                                                                                                                    |
| **Associated visualization documents**   | Real-time monitoring panel; electrical consumption dashboard; device status (visual indicator: online/offline); historical measurement log; configuration of linked devices                                                                                                                  |
| **Input**                                | Data sent by the device: voltage, current, active power, accumulated energy, unique device identifier, authentication key, measurement timestamp                                                                                                                                             |
| **Output**                               | Consumption data stored in the system; device linked to a project; updated device status (online/offline); data-reception confirmation                                                                                                                                                       |
| **Description**                          | The system must manage communication, linking, and monitoring of electrical consumption measurement devices. This includes automatic data reception, secure device authentication, and real-time monitoring of connection status.                                                            |
| **Abnormal situation handling**          | Data transmission failure: automatic retry by the device. Incomplete or invalid data: rejection and error logging. Device authentication error: deny linking or transmission. Device with no connection: mark as "offline" after a set time. Send interval not met: generate a system alert. |

**FUNCTIONAL REQUIREMENTS SKETCH**

The system mockups/prototypes, for both the web and mobile versions, are located in the "Mockups" folder.

## 3.3 Non-Functional Requirements

**Non-Functional Requirements:**

**NFR:** Non-Functional Requirement

**NFRS:** Non-Functional Requirement Specification

**NFR1: Performance** — NFRS1.1: Response time, load capacity, throughput.

**NFR2: Scalability** — NFRS2.1: Ability to handle an increase in users or data.

**NFR3: Security** — NFRS3.1: Protection against unauthorized access, encryption, regulatory compliance.

**NFR4: Usability** — NFRS4.1: Ease of use, accessibility, user experience.

**NFR5: Availability** — NFRS5.1: Percentage of time the system must be operational (e.g., 99.9%).

**NFR6: Maintainability** — NFRS6.1: Ease of updating or fixing the system.

**NFR7: Portability** — NFRS7.1: Ability to operate on different platforms or environments.

### NFR1 Specification Sheet

| Field                                    | Details                                                                                                                                                                     |
| ---------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Identifier**                           | RNF1                                                                                                                                                                        |
| **Name**                                 | System Performance                                                                                                                                                          |
| **Type**                                 | Non-functional                                                                                                                                                              |
| **Requirement that uses/specializes it** | RF2, RF5                                                                                                                                                                    |
| **Critical?**                            | Yes                                                                                                                                                                         |
| **Development priority**                 | High                                                                                                                                                                        |
| **Associated visualization documents**   | Load testing, concurrent-user simulations                                                                                                                                   |
| **Input**                                | Consumption queries, alert generation, chart display                                                                                                                        |
| **Output**                               | Responses within an adequate time (< 2 sec), with no loading failures                                                                                                       |
| **Description**                          | The system must maintain a response time of less than 2 seconds when loading data and charts, support multiple concurrent users, and maintain stable throughput under load. |
| **Abnormal situation handling**          | Performance degradation: activate load balancing or scale services. Congestion: notify the user and retry automatically.                                                    |

### NFR2 Specification Sheet

| Field                                    | Details                                                                                                                                                    |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Identifier**                           | RNF2                                                                                                                                                       |
| **Name**                                 | System Scalability                                                                                                                                         |
| **Type**                                 | Non-functional                                                                                                                                             |
| **Requirement that uses/specializes it** | RF2, RF4, RF6                                                                                                                                              |
| **Critical?**                            | Yes                                                                                                                                                        |
| **Development priority**                 | Medium                                                                                                                                                     |
| **Associated visualization documents**   | Scalable architecture diagram                                                                                                                              |
| **Input**                                | Increase in users or devices                                                                                                                               |
| **Output**                               | System operating without loss of performance                                                                                                               |
| **Description**                          | The system must be able to scale horizontally to handle steady growth in users, connected devices, and data volume, without affecting the user experience. |
| **Abnormal situation handling**          | Service saturation: auto-scaling or task redistribution.                                                                                                   |

### NFR3 Specification Sheet

| Field                                    | Details                                                                                                                                                                                                      |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Identifier**                           | RNF3                                                                                                                                                                                                         |
| **Name**                                 | Information Security                                                                                                                                                                                         |
| **Type**                                 | Non-functional                                                                                                                                                                                               |
| **Requirement that uses/specializes it** | RF1, RF3                                                                                                                                                                                                     |
| **Critical?**                            | Yes                                                                                                                                                                                                          |
| **Development priority**                 | High                                                                                                                                                                                                         |
| **Associated visualization documents**   | Security policy, penetration testing                                                                                                                                                                         |
| **Input**                                | User data, authentication, configuration                                                                                                                                                                     |
| **Output**                               | Encrypted data, access restricted to authorized users only                                                                                                                                                   |
| **Description**                          | The system must protect user information through secure authentication mechanisms, data encryption in transit and at rest, and compliance with regulations such as Colombian Law 1581 of 2012 (Habeas Data). |
| **Abnormal situation handling**          | Unauthorized access attempts: temporary lockout and event logging. Detected vulnerabilities: immediate patching.                                                                                             |

### NFR4 Specification Sheet

| Field                                    | Details                                                                                                                                                 |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Identifier**                           | RNF4                                                                                                                                                    |
| **Name**                                 | Interface Usability                                                                                                                                     |
| **Type**                                 | Non-functional                                                                                                                                          |
| **Requirement that uses/specializes it** | RF1, RF2, RF4, RF5                                                                                                                                      |
| **Critical?**                            | No                                                                                                                                                      |
| **Development priority**                 | Medium                                                                                                                                                  |
| **Associated visualization documents**   | UI/UX prototypes, user testing                                                                                                                          |
| **Input**                                | User interaction with the system                                                                                                                        |
| **Output**                               | Smooth, intuitive experience                                                                                                                            |
| **Description**                          | The application must be easy to use, accessible to users without technical experience, and compliant with usability and accessibility standards (WCAG). |
| **Abnormal situation handling**          | Navigation confusion: contextual assistance or tutorial. User errors: clear messages and correction options.                                            |

# 4. Use Case Requirements

## 4.1 UML Use Case Diagram

The overall diagram, as well as the individual diagrams for each use case, can be found in the "Use Case Diagrams" folder.

## 4.2 Use Case Characterization

### Use Case No. 1 — User Management

| Field            | Details                                                                                                                                                                                                                                                                                 |
| ---------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Name**         | User Management                                                                                                                                                                                                                                                                         |
| **Description**  | The user can register their account directly on the website or mobile application, or alternatively register through social-network providers. The user must accept the terms and permissions corresponding to their user type, enabling data processing and use of the system license. |
| **Priority**     | High                                                                                                                                                                                                                                                                                    |
| **Precondition** | The user must not already be registered, and as the user interacts with the system they must accept the terms and permissions for use of the license.                                                                                                                                   |

**Normal sequence**

| Step | Action                        | System                                                                                              |
| ---- | ----------------------------- | --------------------------------------------------------------------------------------------------- |
| 1    | Enters personal data          | Displays a form with required and optional fields.                                                  |
| 2    | Submits the form              | Verifies that the data is valid and the email is not duplicated.                                    |
| 3    | Confirmation                  | Creates the account, stores the data, and notifies success.                                         |
| 4    | Accepts terms and permissions | The system sends the user a message to enable use of the license.                                   |
| 5    | Updates profile               | The system displays a panel with previously recorded user information.                              |
| 6    | Recovers password             | Validates the user's identity via a secure method in order to change the password.                  |
| 7    | Deletes account               | The system offers the option to delete the account if the user no longer wishes to use the service. |

**Postcondition:** The user is registered, can log in, and can use the license.

**Exceptions (alternate flow)**

| Step | Action                                              |
| ---- | --------------------------------------------------- |
| 1    | Email already registered.                           |
| 2    | Incomplete or invalid fields.                       |
| 3    | The user does not accept the terms and permissions. |

**Actors:** User; Social network provider

**Comments:** Includes secure password validation.

### Use Case No. 2 — Project Management

| Field            | Details                                                                                                                   |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------- |
| **Name**         | Project Management                                                                                                        |
| **Description**  | Allows the responsible user to register projects and devices, link devices to the project, and link users to the project. |
| **Priority**     | Medium                                                                                                                    |
| **Precondition** | The user must be registered as a responsible user.                                                                        |

**Normal sequence**

| Step | Action                 | System                                                                     |
| ---- | ---------------------- | -------------------------------------------------------------------------- |
| 1    | Register project       | The system creates a project where the user can link all its dependencies. |
| 2    | Register devices       | The system captures information about registered devices.                  |
| 3    | Link device to project | The system links devices to the project.                                   |
| 4    | Link users to project  | The system links users (regular users) to the project.                     |

**Postcondition:** The responsible user has created a project with all its dependencies.

**Exceptions (alternate flow)**

| Step | Action                              |
| ---- | ----------------------------------- |
| 1    | The regular user is not registered. |
| 2    | Device not detected.                |

**Actors:** Responsible user; Regular user

**Comments:** It is recommended to verify that users and devices are registered in the system.

### Use Case No. 3 — System Management and Configuration

| Field            | Details                                                                                                             |
| ---------------- | ------------------------------------------------------------------------------------------------------------------- |
| **Name**         | System Management and Configuration                                                                                 |
| **Description**  | The user customizes the system according to their needs.                                                            |
| **Priority**     | High                                                                                                                |
| **Precondition** | The user must have accepted the terms and permissions corresponding to their user type in order to use the license. |

**Normal sequence**

| Step | Action                         | System                                                                                     |
| ---- | ------------------------------ | ------------------------------------------------------------------------------------------ |
| 1    | Change threshold configuration | The system takes in the user's information, adjusting thresholds according to their needs. |
| 2    | Change language                | The system language is configured according to the user's requirements.                    |
| 3    | Modify alert delivery settings | The system communicates with the user through the channel they established.                |
| 4    | Modify colors                  | The system adjusts the user interface, applying the themes set by the user.                |
| 5    | Configure default values       | The system resets to its default values.                                                   |

**Postcondition:** System configuration executed.

**Exceptions (alternate flow)**

| Step | Action                                                                        |
| ---- | ----------------------------------------------------------------------------- |
| 1    | The user mistakenly enters an incorrect value in the threshold configuration. |
| 2    | The user does not save the changes.                                           |

**Actors:** User, System

**Comments:** The system indicates to the user when changes have been applied.

### Use Case No. 4 — Recording and Processing of Electrical Consumption

| Field            | Details                                                                                                                                                                         |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Name**         | Recording and Processing of Electrical Consumption                                                                                                                              |
| **Description**  | Allows the user to check current consumption and compare it with historical electrical consumption, and to receive alerts and recommendations based on the current measurement. |
| **Priority**     | High                                                                                                                                                                            |
| **Precondition** | The user must be associated with a project.                                                                                                                                     |

**Normal sequence**

| Step | Action                                                    | System                                                                              |
| ---- | --------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| 1    | Check the consumption measurement for the current period. | The system loads the consumption information for the current period.                |
| 2    | Access the history section.                               | Loads consumption data from the database.                                           |
| 3    | Select a date range.                                      | Shows charts and tables with itemized consumption.                                  |
| 4    | View alerts.                                              | The system displays alerts based on current consumption behavior.                   |
| 5    | View recommendations.                                     | The system displays recommendations responding to the current measurement behavior. |

**Postcondition:** The user views the consumption history in graphical and tabular format, as well as alerts and recommendations.

**Exceptions (invalid range)**

| Step | Action                                |
| ---- | ------------------------------------- |
| 1    | Selection of an incorrect date range. |

**Actors:** User, System

**Comments:** Interactive charts; report export is optional.

### Use Case No. 5 — Measurement Device Management

| Field            | Details                                                                                                                                                                                               |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Name**         | Measurement Device Management                                                                                                                                                                         |
| **Description**  | Enables automatic transmission of electrical consumption data from the device to the system, its secure linking, monitoring of its connection status, and automatic reconnection in case of failures. |
| **Priority**     | High                                                                                                                                                                                                  |
| **Precondition** | The device must be correctly configured; must have an internet connection (WiFi); must be registered or ready to be linked to the system.                                                             |

**Normal sequence**

| Step | Action                                                                    | System                                                   |
| ---- | ------------------------------------------------------------------------- | -------------------------------------------------------- |
| 1    | The device powers on and connects to the internet.                        | The system becomes available to receive data.            |
| 2    | The device sends its identifier and key.                                  | The system validates and authenticates the device.       |
| 3    | The device is linked to a project.                                        | The system records the association.                      |
| 4    | The device sends data every 60 seconds (voltage, current, power, energy). | The system receives, validates, and stores the data.     |
| 5    | The system analyzes the last measurement received.                        | Determines whether the device is online.                 |
| 6    | The device loses connection (if this occurs).                             | The system detects inactivity.                           |
| 7    | The device reconnects automatically.                                      | The system restores communication and continues logging. |

**Postcondition:** The device is correctly linked to the system; consumption data is stored periodically; the device status remains updated (online/offline).

**Exceptions (alternate flow)**

| Step | Action                                      |
| ---- | ------------------------------------------- |
| 1    | Device already linked.                      |
| 2    | Loss of connection.                         |
| 3    | Data not received within the expected time. |

**Actors:** Measurement device; System

**Comments:**

- Transmission occurs via HTTP protocol every 60 seconds.
- Security must be guaranteed in device authentication.
- The system must be tolerant of network failures.
- Automatic reconnection is the device's responsibility (ERF5.4).
