🟦 Crew Experience Points Engine
Technical Specification + PRD + System Flow (v2.0)

Date: 2025-10-27
Target Audience: Senior .NET Developer / Database Architect
Status: Confirmed for Implementation

1. Business Overview

The Crew Experience Points Engine calculates the experience value of officers only based on their cumulative months served with the same Vessel Manager (Principal). The metric supports:

Manager vetting (RightShip, OCIMF, TMSA-aligned KPIs)

Internal CAMII officer development KPIs

Vessel-level manpower stability assessments

Experience Points (XP) are dynamic and depend on the AsOfDate selected by the module.

1.1 Core Formula (Confirmed)
Cumulative Months with Same Manager	Points	Description
0 to 5 months	1	Entry-level
6 to 11 months	2	Developing
12 to 23 months	3	Experienced
24+ months	4	Senior / Max

Rank Scope: Officers only (RankTypeName = 'Officer')
Outputs:

OfficerXP (per officer)

TotalOfficerXP (sum per vessel)

1.2 AsOfDate Logic (Final)
Module	AsOfDate
Crew Profile	SignOffDate if exists, otherwise Today
Vessel Dashboard	Today
Utilities Reports	@ReportDate parameter

This ensures consistent reporting logic and avoids double-scoring.

2. Logic & Algorithms
2.1 Preset Priority Hierarchy

The XP engine retrieves the active rule preset using this order:

Vessel-Specific Preset

Manager (Principal) Preset

System Default Preset

Effective rule = highest priority available.

2.2 Experience Calculation Logic
Step 1: Identify Manager
SELECT PrincipalId 
FROM PrincipalVessel 
WHERE PrincipalVesselId = @PrincipalVesselId

Step 2: Extract Officer’s Sea Service History

Use only records:

belonging to same Manager (PrincipalId)

with SignOn <= AsOfDate

and (SignOff >= AsOfDate OR SignOff IS NULL)

Step 3: Normalize End Date
IF SignOff IS NULL → use AsOfDate  
IF SignOff > AsOfDate → use AsOfDate  
ELSE use SignOff

Step 4: Compute Months
MonthsWithManager = SUM(DATEDIFF(MONTH, SignOn, EffectiveEnd))

Step 5: Apply Active Preset Bands

Match MonthsWithManager against bands in CrewExperienceRuleBand.

Result per officer → OfficerXP

Result per vessel → SUM(OfficerXP)

3. Database Schema (SQL Server)
3.1 Configuration Tables
dbo.CrewExperienceRulePreset
CREATE TABLE dbo.CrewExperienceRulePreset (
    CrewExperienceRulePresetId INT IDENTITY(1,1) PRIMARY KEY,
    RuleName                   VARCHAR(100) NOT NULL,
    Description                VARCHAR(MAX) NULL,
    IsDefault                  BIT NOT NULL DEFAULT(0),
    IsActive                   BIT NOT NULL DEFAULT(1),
    UserCloudId                INT NULL
);

dbo.CrewExperienceRuleBand
CREATE TABLE dbo.CrewExperienceRuleBand (
    CrewExperienceRuleBandId   INT IDENTITY(1,1) PRIMARY KEY,
    CrewExperienceRulePresetId INT NOT NULL,
    FromMonths                 INT NOT NULL,
    ToMonths                   INT NULL,
    Points                     TINYINT NOT NULL,
    SortOrder                  INT NOT NULL,
    FOREIGN KEY (CrewExperienceRulePresetId)
        REFERENCES dbo.CrewExperienceRulePreset (CrewExperienceRulePresetId)
);

3.2 Assignment Tables (Override System)
Manager-Level Preset
CREATE TABLE dbo.PrincipalExperienceRulePreset (
    PrincipalExperienceRulePresetId INT IDENTITY(1,1) PRIMARY KEY,
    PrincipalId                     INT NOT NULL,
    CrewExperienceRulePresetId      INT NOT NULL,
    IsActive                        BIT NOT NULL DEFAULT(1),
    EffectiveFrom                   DATE NULL,
    EffectiveTo                     DATE NULL
);

Vessel-Level Preset
CREATE TABLE dbo.PrincipalVesselExperienceRulePreset (
    PrincipalVesselExperienceRulePresetId INT IDENTITY(1,1) PRIMARY KEY,
    PrincipalVesselId                     INT NOT NULL,
    CrewExperienceRulePresetId            INT NOT NULL,
    IsActive                              BIT NOT NULL DEFAULT(1),
    EffectiveFrom                         DATE NULL,
    EffectiveTo                           DATE NULL
);

3.3 History Table (Optional Snapshot)
CREATE TABLE dbo.CrewExperienceHistory (
    CrewExperienceHistoryId      INT IDENTITY(1,1) PRIMARY KEY,
    CrewInfoId                   INT NOT NULL,
    AsOfDate                     DATE NOT NULL,
    PrincipalId                  INT NULL,
    PrincipalVesselId            INT NULL,
    CrewExperienceRulePresetId   INT NULL,
    OfficerExperiencePoints      INT NOT NULL,
    TotalOfficerExperiencePoints INT NOT NULL,
    CreatedOn                    DATETIME NOT NULL DEFAULT (GETDATE())
);

4. Stored Procedure Requirements
4.1 Preset Resolution (Pseudo-SQL)
-- Input: @PrincipalVesselId, @AsOfDate

WITH RuleHierarchy AS (
    SELECT 1 AS Priority, CrewExperienceRulePresetId
    FROM PrincipalVesselExperienceRulePreset
    WHERE PrincipalVesselId = @PrincipalVesselId AND IsActive = 1

    UNION ALL
    SELECT 2 AS Priority, CrewExperienceRulePresetId
    FROM PrincipalExperienceRulePreset
    WHERE PrincipalId = (SELECT PrincipalId 
                         FROM PrincipalVessel 
                         WHERE PrincipalVesselId = @PrincipalVesselId)
          AND IsActive = 1

    UNION ALL
    SELECT 3 AS Priority, CrewExperienceRulePresetId
    FROM CrewExperienceRulePreset
    WHERE IsDefault = 1 AND IsActive = 1
)
SELECT TOP 1 CrewExperienceRulePresetId
FROM RuleHierarchy
ORDER BY Priority ASC;

5. User Interface & Functional Requirements
5.1 Admin Module: XP Configuration
Features:

Create/Edit/Delete XP Rule Presets

Manage month bands per preset

Assign preset to:

Manager (Principal)

Vessel

Exception list with override indicators

Audit notes per preset

Key Views:

Preset List

Band Editor

Assignment Grid

Override Summary

5.2 Crew Profile: Experience Timeline
For each SeaService row:
Column	Description
Vessel	Vessel joined
Manager	Principal
SignOn/SignOff	Dates
AsOfDate	Using rule: SignOff else Today
MonthsWithManager	Calculated
OfficerXP	Points
Additional Summaries:

By Manager (grouped totals)

By Vessel (grouped totals)

5.3 Vessel Dashboard
Inputs:

Vessel selector

AsOfDate (defaults to Today)

Output:

Active Officers

MonthsWithManager

OfficerXP

TotalOfficerXP

6. Implementation Notes for .NET Team
6.1 Never hardcode XP bands

Always load bands from:

CrewExperienceRulePreset
CrewExperienceRuleBand

6.2 Month Calculation Consistency

Use one method across:

SQL

C# services

Frontend visualizations

Preferred:

DATEDIFF(MONTH)

6.3 Indexing Requirements (mandatory)

Add indexes to ensure fast reporting:

SeaService (CrewInfoId)
SeaService (PrincipalVesselId)
SeaService (SignOn)
PrincipalVessel (PrincipalId)

6.4 API Endpoints Needed

GET /xp/preset/{id}

GET /xp/preset/resolve?vesselId=&asOfDate=

GET /xp/calculate/vessel/{id}?asOfDate=

GET /xp/calculate/crew/{id}

POST /xp/history/save

7. System Flow / Process Flow
7.1 High-Level System Flow (Textual)

User selects Vessel & AsOfDate

System fetches the vessel’s PrincipalId

System resolves preset using priority logic

System loads preset bands

System identifies all officers on vessel at AsOfDate

For each officer:

Fetch SeaService with same manager

Normalize SignOff → EffectiveEnd

Compute MonthsWithManager

Determine XP band

Sum results → TotalOfficerXP

Return data to UI

UI displays:

Officer XP Table

Manager Summary

Vessel Summary

7.2 Mermaid System Flow Diagram (Markdown)
flowchart TD

A[Vessel Dashboard / Crew Profile / Utilities] --> B[Input: PrincipalVesselId + AsOfDate]
B --> C[Fetch PrincipalId]
C --> D[Resolve Preset (Vessel > Manager > Default)]
D --> E[Load XP Bands]
E --> F[Identify Officers Onboard at AsOfDate]
F --> G[For Each Officer: Fetch SeaService with Same Manager]
G --> H[Normalize SignOff -> EffectiveEnd]
H --> I[Compute MonthsWithManager]
I --> J[Map to XP Band using Preset]
J --> K[Compute OfficerXP]
K --> L[Sum TotalOfficerXP]
L --> M[Return JSON to UI]
M --> N[Render XP Table + Summaries]

8. Acceptance Criteria
Functional

Officers only.

XP must always reflect correct AsOfDate logic.

Preset resolution must always follow priority.

UI must show timeline + manager summary + vessel summary.

Non-Functional

SQL queries must complete < 150 ms for 5,000 sea-service rows.

API must be stateless and cache default presets.

XP calculation must be consistent across modules.

9. Versioning Reference
Version	Notes
1.0	Initial concept
2.0	Finalized schema, logic, UI, flows (this document)