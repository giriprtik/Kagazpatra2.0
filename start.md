The Nagarik Vault architecture is well-scoped for a Nepali government pilot. Separating the Spec (regulatory rules) from the Visual Template (print geometry), enforcing the 4-layer decomposition, and maintaining strict boundaries around "extracted and consistency-checked" vs. "authentic" directly address the technical, legal (Individual Privacy Act, 2075), and institutional scrutiny of agencies like DoNIDCR, DoP, and DoTM.

1. System Architecture & Repository Structure

To ensure clean separation between business/audit logic (Java) and computer vision/OCR (Python), organize the project as a monorepo with strict internal API boundaries:
Show Code
Download diagram SVG

2. Core Data Schemas & Spec Design


A. Versioned Document Spec (specs/citizenship/v1.0.0.json)

Each spec captures every historical and modern variant of a document, tagging fields by the print_versions they apply to so the UI dynamically renders only relevant inputs.
json
{
  "spec_id": "np.gov.citizenship",
  "spec_version": "1.0.0",
  "document_type": "CITIZENSHIP_CERTIFICATE",
  "legal_basis": [
    "Nepal Citizenship Act, 2063 (2006)",
    "Nepal Citizenship Rules, 2063 (Schedule 1, 2, 7)"
  ],
  "print_versions": [
    "CIT_V1_HANDWRITTEN_PRE_2063",
    "CIT_V2_PRINTED_2063_SINGLE_LANG",
    "CIT_V3_PRINTED_BILINGUAL_NEP_ENG"
  ],
  "fields": [
    {
      "key": "citizenship_number",
      "label_ne": "नागरिकता प्रमाणपत्र नं.",
      "label_en": "Citizenship Certificate No.",
      "versions": ["CIT_V1_HANDWRITTEN_PRE_2063", "CIT_V2_PRINTED_2063_SINGLE_LANG", "CIT_V3_PRINTED_BILINGUAL_NEP_ENG"],
      "required": true,
      "pii_encrypted": true,
      "format_regex": "^[0-9०-९]{2}-[0-9०-९]{2}-[0-9०-९]{2}-[0-9०-९]{5}$|^[0-9०-९]+(/[0-9०-९]+)*$",
      "normalization": "CONVERT_DEVANAGARI_DIGITS_TO_ASCII"
    },
    {
      "key": "full_name_ne",
      "label_ne": "नाम, थर",
      "label_en": "Full Name (Devanagari)",
      "versions": ["CIT_V1_HANDWRITTEN_PRE_2063", "CIT_V2_PRINTED_2063_SINGLE_LANG", "CIT_V3_PRINTED_BILINGUAL_NEP_ENG"],
      "required": true,
      "pii_encrypted": true
    },
    {
      "key": "full_name_en",
      "label_ne": "Full Name",
      "label_en": "Full Name (English)",
      "versions": ["CIT_V3_PRINTED_BILINGUAL_NEP_ENG"],
      "required": true,
      "pii_encrypted": true
    },
    {
      "key": "birth_district",
      "label_ne": "जन्म स्थान (जिल्ला)",
      "label_en": "Birth District",
      "versions": ["CIT_V1_HANDWRITTEN_PRE_2063", "CIT_V2_PRINTED_2063_SINGLE_LANG", "CIT_V3_PRINTED_BILINGUAL_NEP_ENG"],
      "required": true,
      "pii_encrypted": false,
      "lookup_table": "NEPAL_DISTRICTS_77"
    },
    {
      "key": "birth_ward_no",
      "label_ne": "वडा नं.",
      "label_en": "Birth Ward No.",
      "versions": ["CIT_V1_HANDWRITTEN_PRE_2063", "CIT_V2_PRINTED_2063_SINGLE_LANG", "CIT_V3_PRINTED_BILINGUAL_NEP_ENG"],
      "required": true,
      "pii_encrypted": false,
      "cross_field_rule": "WARD_WITHIN_LOCAL_LEVEL_MAX"
    },
    {
      "key": "dob_bs",
      "label_ne": "जन्म मिति (वि.सं.)",
      "label_en": "Date of Birth (BS)",
      "versions": ["CIT_V1_HANDWRITTEN_PRE_2063", "CIT_V2_PRINTED_2063_SINGLE_LANG", "CIT_V3_PRINTED_BILINGUAL_NEP_ENG"],
      "required": true,
      "pii_encrypted": true,
      "format_regex": "^(19|20)[0-9]{2}-(0[1-9]|1[0-2])-(0[1-9]|[12][0-9]|3[0-2])$"
    }
  ]
}
Use code with caution.

B. Visual Template (visual-templates/citizenship/printed-new-front/template.json)

Defines physical geometry relative to a normalized canonical resolution (e.g., 2000 x 1400 px):
• Reference Image: Blank canonical scan (template_blank.png) for Layer 1 subtraction/alignment.
• Label Anchors: Fixed text coordinates (e.g., "नेपाल सरकार", "गृह मन्त्रालय", "नागरिकता प्रमाणपत्र") used for homography/thin-plate spline alignment.
• Field Zones: Normalized [x1, y1, x2, y2] bounding boxes mapped to spec.fields[].key.
• Layer 3 Zones: Expected polygons for photo, thumbprints (left/right), issuing officer signature, and circular office stamp.

3. Database Schema (PostgreSQL + Flyway)

Enforces Privacy Act 2075 data minimisation, cryptographic hashing, field-level encryption, and an append-only audit trail.
sql
-- V1__init_nagarik_vault.sql

CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255), -- Nullable if Google OAuth only
    oauth_provider VARCHAR(50),
    oauth_subject VARCHAR(255),
    accepted_terms_version VARCHAR(32) NOT NULL,
    accepted_terms_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE documents (
    id UUID PRIMARY KEY,
    owner_user_id UUID NOT NULL REFERENCES users(id),
    document_type VARCHAR(64) NOT NULL,         -- CITIZENSHIP, PASSPORT, DRIVING_LICENCE
    detected_version VARCHAR(64),               -- e.g. CIT_V3_PRINTED_BILINGUAL_NEP_ENG
    spec_version VARCHAR(32) NOT NULL,
    visual_template_id VARCHAR(64),
    status VARCHAR(32) NOT NULL,                -- REJECTED_QUALITY, PROCESSING, NEEDS_REVIEW, VERIFIED_BY_USER
    disclaimer_status VARCHAR(64) NOT NULL DEFAULT 'EXTRACTED_AND_CONSISTENCY_CHECKED',
    
    -- Data Minimisation: Explicit S3/MinIO object keys only
    original_object_key VARCHAR(512) NOT NULL,
    original_sha256 CHAR(64) NOT NULL,
    layer1_template_key VARCHAR(512),
    layer2_pii_mask_key VARCHAR(512),
    layer3_stamps_sigs_key VARCHAR(512),
    layer4_anomalies_key VARCHAR(512),
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Encrypted per-field storage + revision history
CREATE TABLE document_fields (
    id UUID PRIMARY KEY,
    document_id UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    field_key VARCHAR(128) NOT NULL,
    encrypted_value BYTEA,                      -- AES-256-GCM encrypted
    plain_value_non_pii VARCHAR(255),           -- Only for non-PII lookup values (e.g., District) if needed
    ocr_confidence NUMERIC(5,4),
    validation_status VARCHAR(32) NOT NULL,     -- VALID, INVALID_FORMAT, CROSS_CHECK_MISMATCH
    validation_message TEXT,
    human_verified BOOLEAN NOT NULL DEFAULT FALSE,
    verified_by UUID REFERENCES users(id),
    verified_at TIMESTAMPTZ,
    revision INT NOT NULL DEFAULT 1,
    UNIQUE(document_id, field_key, revision)
);

-- Unmapped detections queue (Data Minimisation rule)
CREATE TABLE unmapped_artifacts (
    id UUID PRIMARY KEY,
    document_id UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    crop_object_key VARCHAR(512) NOT NULL,
    detected_text TEXT,
    bounding_box JSONB NOT NULL,
    resolution_status VARCHAR(32) NOT NULL DEFAULT 'PENDING', -- PENDING, ASSIGNED_FIELD, MOVED_TO_LAYER4, DISCARDED
    assigned_field_key VARCHAR(128),
    resolved_by UUID REFERENCES users(id),
    resolved_at TIMESTAMPTZ
);

-- Immutable Audit Log (Every state change, read, export, delete)
CREATE TABLE audit_logs (
    id BIGSERIAL PRIMARY KEY,
    actor_user_id UUID REFERENCES users(id),
    document_id UUID,                           -- Not FK so logs survive document deletion
    action VARCHAR(64) NOT NULL,                -- UPLOAD, QUALITY_REJECT, EXTRACT, FIELD_EDIT, HUMAN_VERIFY, VIEW_PII, EXPORT, DELETE, ARTIFACT_DISCARD
    metadata JSONB NOT NULL,                    -- Before/after hashes, IP, user-agent, reason
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
Use code with caution.

4. Pipeline Execution & Validation Rules


Step-by-Step Flow (Spring Boot Orchestrator \(\leftrightarrow \) Python NepDoc Service)

1. Upload & Hashing: Spring Boot streams upload to MinIO/S3 while computing SHA-256 on the fly. Saves record and calls POST /internal/v1/process on the Python worker.
2. Quality Gate (Python):
	• Blur: Laplacian variance < 100.0 → Reject ("Image is too blurry; hold steady and tap to focus").
	• Glare: HSV saturation/value hotspot ratio > 3% over text zones → Reject ("Glare detected on laminated surface; tilt document away from direct light").
	• Resolution: Effective DPI < 200 (width < 1200px for ID cards) → Reject ("Resolution too low; move camera closer").
	• Framing/Orientation: Contour area < 60% of frame or clipped corners → Reject ("Edges cut off; ensure all four corners are visible").
3. Classification & Alignment:
	• Classifies Document Type + Side + Print Version (CIT_V3_FRONT, PASSPORT_MRP, PASSPORT_EPASSPORT, DL_SMART_CARD).
	• Applies perspective dewarp + feature/label-anchor homography against the matched visual-template.
4. 4-Layer Separation:
	• Layer 1: Registered blank form template + matched structural lines.
	• Layer 2: Cropped personal text zones, portrait photo, and fingerprint blocks (written directly to restricted S3 prefix; never returned in unencrypted logs).
	• Layer 3: Ink/color segmentation extracting red/blue/purple circular office stamps and handwritten signatures, ordered top-to-bottom / left-to-right by application sequence.
	• Layer 4: Residual diff image (Original_Aligned - (Layer1 + Layer2 + Layer3)) capturing holograms, lamination peeling, punch holes, or margin notes.
5. Deterministic Validators (Java Domain Service):
	• ICAO 9303 MRZ Check Digits (Passport): Computes weighted 7-3-1 modulo-10 checksums across Passport Number, DOB, Expiry Date, Personal Number, and Composite Check Digit.
	• Nepali Administrative Hierarchy: Validates District ∈ 77 districts, Local Level ∈ 753 municipalities/rural municipalities belonging to that district, and 1 <= Ward <= max_wards(local_level).
	• BS \(\leftrightarrow \) AD Calendar Cross-Check: Uses a deterministic Bikram Sambat lookup table (1970 BS – 2100 BS) to verify that dob_bs and dob_ad (e.g., on bilingual citizenship cards or across Citizenship vs. Passport) resolve to the exact same calendar day.
	• Cross-Document Consistency: Flags mismatches when a user's Citizenship Name/DOB/Citizenship No. conflicts with their Passport or Driving Licence.

5. Pilot Guardrails & Reviewer Defensibility

To ensure zero overclaiming during government evaluations:
• UI Banner & Watermark: Every document view and exported JSON/PDF report displays a permanent status badge: EXTRACTED & CONSISTENCY-CHECKED — NOT AUTHENTICATED BY ISSUING AUTHORITY.
• Layer 3 Semantics: Stamp checks output factual geometric/color metrics (e.g., "Circular red ink impression detected in Issuing Officer zone; text matches 'जिल्ला प्रशासन कार्यालय'") rather than "Genuine Stamp".
• Human-in-the-Loop Gate: A document cannot transition to VERIFIED_BY_USER until every required field and every unmapped_artifact has an explicit human confirmation logged in audit_logs.
• Air-Gapped Processing: docker-compose blocks outbound egress on the processor container network so no PII can ever reach external APIs.

Recommended Build Order (Milestones)

1. Milestone 1 (Foundation & Compliance): Docker Compose (Postgres, MinIO, Spring Boot, React, Nginx), Flyway schema, Email/Google OAuth signup with versioned Privacy Terms acceptance, and Audit Log interceptor.
2. Milestone 2 (Specs & Deterministic Rule Engine): JSON specs for Citizenship, Passport, and Driving Licence + Java unit test suite for ICAO 9303 MRZ check digits, 77/753/ward validation, and BS–AD date conversion.
3. Milestone 3 (Python NepDoc Integration): Internal FastAPI wrapper exposing Quality Gate, Version Classifier, Homography Alignment, 4-Layer Splitter, and Tesseract + Surya OCR.
4. Milestone 4 (Verification UI & Camera Capture): getUserMedia edge-detecting scanner, 4-layer visual inspector, side-by-side human field verification, and Unmapped Artifact review queue.
To start building immediately, which artifact would you like first:
• Complete JSON Specs (Citizenship, Passport with ICAO 9303 rules, Driving Licence) + Java validation & audit codebase
• Python FastAPI NepDoc wrapper (Quality gate, homography alignment, 4-layer extraction, Surya + Tesseract)
• Docker Compose & infrastructure setup (local + AWS templates, Nginx, Flyway migrations, Spring Security + Terms consent)
