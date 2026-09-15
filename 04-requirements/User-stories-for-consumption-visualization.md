### Functional User Stories

#### User Story 1: Registration and Account Management

- **Title:** Registration, authentication, and user profile management

- **As a:** system user (web or mobile)

- **I want:** to register, authenticate (directly or via social networks), edit my profile, recover my password, and accept the terms and licenses

- **So that:** I can securely access the platform and manage my projects and energy consumption.

- **Acceptance Criteria:**
  1.  Allows manual registration with name, email, password, and the option of authentication/registration via social network providers.

  2.  Validates that the email address has not been previously registered.

  3.  Includes explicit acceptance of the license's terms, conditions, and permissions.

  4.  Allows logging in, updating profile data, requesting a password reset via secure email, and deleting the account.

- **Priority:** High

- **Dependencies:** Base for RF2, RF3, RF4, and RF5.

#### User Story 2: Project and Home Management

- **Title:** Creation of projects/homes and linking of dependencies

- **As a:** responsible user

- **I want:** to register projects or homes, link regular users, and assign measurement devices

- **So that:** I can structure the administration of energy consumption for a specific environment and grant the corresponding permissions.

- **Acceptance Criteria:**
  1.  The responsible user can register a project or home by indicating name, location, and description.

  2.  Allows associating secondary (regular) users with the project.

  3.  Assigns advanced permissions to the responsible user for managing the environment and its devices.

  4.  Any user with access can mark projects or homes as favorites.

- **Priority:** High

- **Dependencies:** RF1 (User Management). Uses RF3 and RF4.

#### User Story 3: System Configuration and Customization

- **Title:** Customization of thresholds, alerts, and interface

- **As a:** system user

- **I want:** to adjust consumption thresholds, notification channels, color palette, and language

- **So that:** I can adapt the application to my visual, language, and energy-control needs.

- **Acceptance Criteria:**
  1.  Allows defining custom consumption thresholds (hourly, daily, or monthly) or keeping the system's default values.

  2.  The user can choose the alert delivery method (push notifications in the app, email, or both).

  3.  Allows selecting among color palettes that adapt to dark mode and changing the system language (up to 4 languages).

  4.  Validates that the values entered for the thresholds are valid.

- **Priority:** Medium-High

- **Dependencies:** RF1 (User Management). Uses RF4.

#### User Story 4: Query and Analysis of Electrical Consumption

- **Title:** Visualization, comparison, and trends of consumption

- **As a:** registered user associated with a project

- **I want:** to view my historical measurements, filter by date ranges, and see trend charts

- **So that:** I can identify consumption patterns, peaks, or critical time periods and evaluate energy-saving opportunities.

- **Acceptance Criteria:**
  1.  Displays recorded consumption in dynamic charts (trends, peaks, critical time periods) and tables.

  2.  Allows filtering the query by custom date ranges.

  3.  Allows comparing current consumption against historical averages.

  4.  Allows exporting or generating reports in PDF format.

- **Priority:** Very High

- **Dependencies:** RF1, RF2, RF3, and RF5.

#### User Story 5: Alert Management and Automatic Recommendations

- **Title:** Notification of excesses and efficiency suggestions

- **As a:** system user

- **I want:** to receive automatic alerts when thresholds are exceeded, along with personalized guidance for efficient use

- **So that:** I can react in a timely manner to unusual consumption and optimize my usage habits.

- **Acceptance Criteria:**
  1.  Generates and reports automatic real-time alerts when excesses are detected relative to the configured thresholds.

  2.  Generates personalized automatic recommendations based on detected consumption behavior.

  3.  Allows viewing the alert history, resolving alerts, and marking recommendations as read.

- **Priority:** High

- **Dependencies:** ERF4.3 and ERF4.4 (included in RF4), RF3 (Configuration).

#### User Story 6: Measurement Device Management

- **Title:** Data transmission, linking, and monitoring of the meter

- **As a:** system / responsible user

- **I want:** the measurement device to send periodic readings, authenticate securely, and report its connection status

- **So that:** I can guarantee the automated and uninterrupted recording of electrical variables.

- **Acceptance Criteria:**
  1.  The device transmits data (voltage, current, active power, and accumulated energy) every 60 seconds via the HTTP protocol.

  2.  The system authenticates and links the device using a unique identifier and access key.

  3.  Displays the device status in real time (_online / offline_).

  4.  The device performs automatic reconnection in the event of WiFi or communication failures.

- **Priority:** Very High

- **Dependencies:** RF1, RF2, RF3, RF5.

### Non-Functional User Stories

#### User Story 7: System Performance

- **Title:** Response time and agile interactivity

- **As a:** frequent user of the application

- **I want:** queries, charts, and alert generation to respond quickly

- **So that:** I can interact fluidly and without delays with the platform.

- **Acceptance Criteria:**
  1.  The response time for loading data and charts is less than 2 seconds.

  2.  Supports multiple concurrent users while maintaining stable throughput.

- **Priority:** High

- **Dependencies:** RNF1 (Applies to RF2 and RF5).

#### User Story 8: System Scalability

- **Title:** System growth capacity

- **As a:** administrator / technical team

- **I want:** the system architecture to scale horizontally

- **So that:** it can support the constant increase in users, connected devices, and data volume without degrading the service.

- **Acceptance Criteria:**
  1.  Supports load spikes and constant data growth through auto-scaling or task redistribution.

  2.  The system operates continuously without loss of performance under increased load.

- **Priority:** Medium

- **Dependencies:** RNF2.

#### User Story 9: Information Security

- **Title:** Data protection and secure authentication

- **As a:** registered user

- **I want:** my credentials and consumption data to be transmitted and stored using encryption mechanisms

- **So that:** I can guarantee the privacy of my information and prevent unauthorized access.

- **Acceptance Criteria:**
  1.  Encryption of data both in transit and at rest.

  2.  Secure authentication mechanisms and temporary lockout upon repeated unauthorized access attempts.

  3.  Compliance with personal data protection policies and regulations.

- **Priority:** High

- **Dependencies:** RNF3 (Applies to RF1 and RF3).

#### User Story 10: Usability and Accessibility

- **Title:** Intuitive and accessible interface

- **As a:** user without specialized technical experience

- **I want:** to have a responsive, clear interface with contextual help

- **So that:** I can navigate and interpret information without complications on mobile devices or the web.

- **Acceptance Criteria:**
  1.  Intuitive, consistent, and responsive design applicable across web and mobile environments.

  2.  Compliance with WCAG accessibility guidelines and standards.

  3.  Clear error messages and contextual assistance or help mechanisms.

- **Priority:** Medium

- **Dependencies:** RNF4 (Applies to RF1, RF2, RF4, and RF5).
