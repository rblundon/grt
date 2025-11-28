# Product Requirements Document: Genealogy Research Tool

## 1. Overview

### 1.1 Product Vision
A containerized, web-based genealogy research tool that automates the extraction and organization of family relationship data from multiple sources, streamlining genealogical research by integrating obituaries, public records, and DNA testing data into a centralized, standards-compliant database.

### 1.2 Goals
- Reduce manual data entry time for genealogical research by 80%+
- Automatically extract and structure relationship data from unstructured sources using LLM technology
- Maintain GEDCOM compliance for data portability
- Build a scalable, cost-effective foundation for multi-source data integration
- Minimize external API costs through intelligent caching

### 1.3 Target User
Individual genealogy researchers who want to accelerate their family history research through automated data extraction and relationship mapping.

## 2. Technical Architecture

### 2.1 Technology Stack
- **Backend**: Python with Gramps Web integration
- **Frontend**: JavaScript framework (React/Vue recommended)
- **Primary Database**: Gramps Web (GEDCOM-compliant genealogical data)
- **Cache Database**: MariaDB (obituary content, LLM responses, extracted entities)
- **LLM Provider**: OpenAI (initial), extensible to support multiple providers
- **Containerization**: Podman with multi-container architecture
- **Data Standard**: GEDCOM format for all genealogical data

### 2.2 Container Architecture

#### Container Services
1. **Frontend Container**: Web UI application
2. **Backend Container**: Python application server with business logic
3. **Gramps Web Container**: Genealogical database and API
4. **MariaDB Container**: Cache database
5. **Network**: All containers on same Podman network for inter-service communication

#### Container Communication
- Frontend ↔ Backend: REST API
- Backend ↔ Gramps Web: Gramps Web REST API
- Backend ↔ MariaDB: Database connection
- Backend ↔ OpenAI: HTTPS API calls

### 2.3 Data Sources
- **Phase 1**: Obituaries (URL-based input)
- **Future Phases**: 23andMe API, TruePeopleSearch.com
- **Lookup Strategy**: APIs preferred, web scraping as fallback

### 2.4 Core Components

#### Input Handler
- Accept and validate obituary URLs
- Check cache before fetching
- Queue processing jobs

#### Content Fetcher
- Fetch obituary HTML from URL
- Extract clean text content
- Store raw HTML and text in MariaDB cache
- Respect robots.txt and rate limits

#### LLM Extractor
- Send obituary text to OpenAI API with structured extraction prompt
- Parse LLM JSON response for entities and relationships
- Calculate confidence scores
- Cache LLM requests and responses in MariaDB
- Track token usage and costs

#### Entity Processor
- Store extracted persons in MariaDB
- Store extracted relationships in MariaDB
- Apply confidence thresholds for auto-store vs. review

#### Gramps Web Connector
- Query existing Gramps records for potential matches
- Create new person records in Gramps Web
- Create family records linking related individuals
- Add source citations (obituary URLs)
- Update MariaDB with Gramps record IDs

#### Cache Manager
- Check cache before external calls (obituary fetch, LLM API)
- Implement cache invalidation policies
- Provide cache statistics and cost tracking

#### Review Interface
- Display extracted entities requiring manual review
- Allow editing of person data and relationships
- Approve or reject automatic matching
- Manually link to existing Gramps records

## 3. Phase 1 Requirements: Obituary Processing

### 3.1 Functional Requirements

#### FR1: URL Input
- Accept obituary URL through web interface
- Validate URL format and accessibility
- Check MariaDB cache for existing entry (by URL hash)
- Support major obituary platforms (Legacy.com, Tributes.com, funeral homes, newspapers)
- Display cache hit notification to user

#### FR2: Content Extraction
- Fetch obituary content from provided URL (if not cached)
- Extract full text of obituary
- Calculate content hash for change detection
- Store raw HTML and extracted text in `obituary_cache` table
- Record fetch timestamp, HTTP status, any errors
- Handle paywalls and access restrictions gracefully with error messaging
- Respect source websites (user-agent, rate limiting, robots.txt)

#### FR3: LLM-Based Entity Extraction
- Check `llm_cache` for existing extraction (by prompt hash)
- If not cached, send obituary text to OpenAI API with structured prompt
- Prompt LLM to return JSON with:
  - List of all persons mentioned
  - Each person's attributes (name, age, dates, locations, gender)
  - All relationships between persons
  - Confidence scores for each extraction
  - Identification of primary deceased individual
- Parse LLM JSON response
- Store in `llm_cache` table with token usage and cost
- Store extracted persons in `extracted_persons` table
- Store relationships in `extracted_relationships` table

**Entity Attributes to Extract:**
- Full name (required)
- Given names, surname, maiden name
- Age (if mentioned)
- Birth date and location (if mentioned, flag if approximate)
- Death date and location (if mentioned, flag if approximate)
- Residence location
- Gender
- Role (primary deceased vs. survivor/relative)

**Relationship Types to Identify:**
- Spouse/partner
- Parent/child (including step-, adoptive)
- Sibling (including half-, step-)
- Grandparent/grandchild
- In-laws
- Other extended family
- Preceded in death by / Survived by context

**Confidence Scoring Criteria:**
- **High confidence (0.85-1.0)**: Clear relationship terms ("survived by his wife Mary Smith"), full names, specific dates
- **Medium confidence (0.60-0.84)**: Ambiguous terms ("Mary was present"), partial information
- **Low confidence (0.0-0.59)**: Unclear mentions, potential ambiguity

#### FR4: Intelligent Matching & Linking
- Query Gramps Web for potential matches based on:
  - Name similarity (exact, phonetic, nicknames)
  - Date ranges (birth/death within reasonable window)
  - Location overlap
  - Existing family connections
- For each extracted person:
  - **High match confidence**: Auto-link to existing Gramps record, set `match_status='matched'`
  - **Medium match confidence**: Flag for review, set `match_status='review_needed'`
  - **No match found**: Mark for creation, set `match_status='unmatched'`
- Store Gramps person IDs in `extracted_persons.gramps_person_id`
- For persons in multiple obituaries: link all obituaries as sources to same Gramps record

#### FR5: Data Storage with Confidence Thresholds
- Check `config_settings` for current thresholds:
  - `confidence_threshold_auto_store` (default: 0.85)
  - `confidence_threshold_review` (default: 0.60)
  - `always_review` flag (default: false)
  
**Storage Logic:**
- If `always_review=true`: All extractions go to review queue
- If extraction confidence ≥ `confidence_threshold_auto_store` AND match confidence is high:
  - Automatically create/update Gramps Web records
  - Set `match_status='created'` or `'matched'`
  - Store mapping in `gramps_record_mapping`
- If extraction confidence < `confidence_threshold_auto_store` OR match confidence is medium:
  - Flag for manual review
  - Set `match_status='review_needed'`
- If confidence < `confidence_threshold_review`:
  - Do not store, log as low-quality extraction

**Gramps Web Storage:**
- Create person records with all extracted attributes
- Create family records for relationships
- Create event records (birth, death, marriage)
- Add source record for obituary with URL citation
- Link all entities to source
- Store bidirectional mapping in MariaDB

#### FR6: Comprehensive Caching System
**Three-Layer Cache:**
1. **Obituary Content Cache** (`obituary_cache` table)
   - Cache raw HTML and extracted text
   - Use URL hash for lookups
   - Include fetch timestamp and content hash
   - Cache hit avoids web scraping

2. **LLM Response Cache** (`llm_cache` table)
   - Cache prompt and response for each LLM call
   - Use prompt hash for deduplication
   - Track provider, model version, tokens, cost
   - Cache hit avoids expensive API calls

3. **Extracted Entity Cache** (`extracted_persons`, `extracted_relationships` tables)
   - Store structured extraction results
   - Enable quick re-processing without LLM calls
   - Support batch operations

**Cache Policies:**
- Obituaries: 365-day expiry (configurable)
- LLM responses: No expiry (prompts rarely change)
- Regular cleanup of stale cache entries
- Track cache hit rate metrics

#### FR7: Manual Review Workflow
- Display list of entities flagged for review (`match_status='review_needed'`)
- For each entity show:
  - Extracted data
  - Confidence scores
  - Potential Gramps matches (if any)
  - Source obituary context
- Allow user to:
  - Edit extracted data
  - Confirm or reject automatic matching
  - Manually select correct Gramps record
  - Create new Gramps record
  - Reject/delete extraction
- After review, update `match_status` and proceed with storage

#### FR8: Cost Tracking & Monitoring
- Track all LLM API usage in `llm_cache`:
  - Prompt tokens
  - Completion tokens
  - Total tokens
  - Estimated cost in USD
- Provide dashboard view:
  - Daily/monthly cost trends
  - Cost per obituary processed
  - Cache hit rate (cost savings)
  - Token usage by model
- Alert if daily cost exceeds threshold (configurable)

### 3.2 Non-Functional Requirements

#### NFR1: Data Quality
- Maintain GEDCOM compliance for all Gramps Web data
- Validate extracted data before storage
- Provide mechanism to review and correct extractions
- Audit trail for all automated actions in `audit_log` table
- Data integrity enforced through foreign keys

#### NFR2: Performance
- Process typical obituary (5-15 people) within 30 seconds (including LLM call)
- Cache checks complete in < 100ms
- Support batch processing via `processing_queue` table
- Concurrent processing of multiple obituaries
- Maximum 3 retry attempts for failed operations

#### NFR3: Cost Efficiency
- Target < $0.10 per obituary processed (LLM costs)
- Cache hit rate target: 70%+ for repeated processing
- Automatic batching of requests when possible
- Use GPT-3.5-turbo for initial extraction, GPT-4 only for complex cases (future enhancement)

#### NFR4: Reliability
- Handle network failures with retry logic (max 3 attempts)
- Graceful degradation when LLM API unavailable
- Clear error messages in UI
- All processing jobs tracked in `processing_queue`
- Failed jobs automatically queued for retry

#### NFR5: Scalability
- Containerized architecture supports horizontal scaling
- Database connection pooling
- Async job processing for batch operations
- Prepared for future data sources (23andMe, TruePeopleSearch)

#### NFR6: Maintainability
- Multi-LLM provider architecture (OpenAI initial, others pluggable)
- Tunable confidence thresholds via `config_settings` table
- Comprehensive logging and audit trails
- Database schema supports schema versioning/migrations

#### NFR7: Security & Privacy
- No authentication in Phase 1 (future enhancement)
- Container network isolation
- Secure storage of API keys (environment variables)
- MariaDB with strong password
- Only publicly available obituary data stored

## 4. Data Model

### 4.1 MariaDB Cache Schema
Comprehensive schema defined in separate artifact including:
- `obituary_cache`: Raw content and metadata
- `llm_cache`: LLM requests/responses with cost tracking
- `extracted_persons`: Identified individuals
- `extracted_relationships`: Family connections
- `gramps_record_mapping`: Links to Gramps Web records
- `config_settings`: Tunable parameters
- `processing_queue`: Async job management
- `audit_log`: Complete action history

### 4.2 Gramps Web Entities

#### Person Record
- Name (primary and alternate)
- Gender
- Birth event (date, place, circa flag)
- Death event (date, place, circa flag)
- Residence
- Source citations (obituary URL via source records)
- Notes (confidence level, extraction metadata)

#### Family Record
- Relationship type (married, partnered, etc.)
- Connected person IDs (parent1, parent2, children)
- Source citations

#### Event Record
- Event type (birth, death, marriage, etc.)
- Date (with circa flag if approximate)
- Place
- Source citations

#### Source Record
- Title (obituary title or "Obituary of [Name]")
- Publication information
- URL
- Repository (newspaper, funeral home, Legacy.com, etc.)

#### Citation Record
- Links source to person/family/event
- Confidence note
- Page/section (if applicable)

## 5. User Interface Requirements

### 5.1 Phase 1 Interface

#### Home/Input Page
- Single input field for obituary URL
- "Process" button
- Checkbox: "Always require manual review before saving"
- Link to view processing queue
- Link to view entities requiring review
- Link to cost tracking dashboard

#### Processing Page
- Progress indicator with stages:
  - Checking cache...
  - Fetching obituary...
  - Extracting entities with AI...
  - Matching with existing records...
  - Storing data...
- Display cache hit notifications ("Using cached obituary - no fetch needed!")
- Show estimated cost for LLM processing
- Real-time status messages

#### Review Page (Conditional)
- Appears only if entities require review
- List of extracted persons with:
  - Full details
  - Confidence scores (visual indicator: green/yellow/red)
  - Potential Gramps matches
  - Source context from obituary
- Edit controls for each person/relationship
- For each potential match:
  - Show existing Gramps record details
  - "Confirm Match" / "Reject Match" / "Create New Record" buttons
- Relationship visualization (simple tree or list)
- "Save All" button to commit approved changes
- "Discard All" button to cancel

#### Results Page
- Success confirmation
- Summary statistics:
  - X persons processed
  - Y new records created
  - Z records linked/updated
  - Cost: $X.XX (tokens used)
- Links to view new records in Gramps Web
- Cache status (hit/miss)
- "Process Another Obituary" button

#### Cost Dashboard
- Current month cost total
- Cost per obituary (average)
- Token usage chart (daily/weekly/monthly)
- Cache hit rate percentage
- LLM provider/model breakdown
- Configurable cost alert threshold

#### Review Queue Page
- List of all entities flagged for review
- Sortable by confidence score, date, obituary
- Batch review capability
- Filters: by confidence range, by match status

### 5.2 Configuration Page (Admin)
- Tunable thresholds:
  - Auto-store confidence threshold (slider: 0.0-1.0)
  - Review confidence threshold (slider: 0.0-1.0)
  - Always require review (checkbox)
  - Cache expiry days
  - Cost alert threshold
- LLM provider settings:
  - Default provider (dropdown: OpenAI, future: Claude, Ollama)
  - Default model
  - API key management
- Gramps Web connection:
  - URL/hostname
  - API credentials
  - Test connection button

## 6. API & Integration Requirements

### 6.1 Gramps Web API Integration
- Authenticate with Gramps Web instance (API token)
- REST API endpoints:
  - `GET /api/people/` - Search existing persons
  - `POST /api/people/` - Create person record
  - `GET /api/families/` - Search existing families
  - `POST /api/families/` - Create family record
  - `POST /api/events/` - Create event records
  - `POST /api/sources/` - Create source records
  - `POST /api/citations/` - Create citations
- Handle API errors and rate limits gracefully
- Batch operations where possible

### 6.2 OpenAI API Integration
- Use `gpt-4-turbo-preview` or `gpt-3.5-turbo` (configurable)
- Structured prompt engineering:
  - Clear instructions for entity extraction
  - JSON output format specification
  - Examples of desired output
  - Confidence scoring instructions
- Function calling or JSON mode for structured output
- Error handling:
  - Rate limit errors (exponential backoff)
  - Invalid API key
  - Model unavailable
  - Timeout (30s limit)
- Token usage tracking for all calls

### 6.3 Web Scraping Requirements
- Respect robots.txt for all domains
- User-agent: "GenealogyResearchBot/1.0 (+URL_TO_YOUR_REPO)"
- Rate limiting: Max 1 request per 2 seconds per domain
- Timeout: 10 seconds for fetch
- Handle common issues:
  - 404 Not Found
  - 403 Forbidden (paywall)
  - SSL certificate errors
  - Redirect loops
  - JavaScript-rendered content (use Playwright/Selenium if needed)
- Clean HTML extraction (BeautifulSoup/lxml)
- Text extraction (remove scripts, styles, navigation)

### 6.4 MariaDB Integration
- Connection pooling (min: 2, max: 10 connections)
- Parameterized queries (prevent SQL injection)
- Transaction support for multi-table operations
- Schema migration tool (Alembic recommended)
- Backup and restore procedures
- Database initialization script

## 7. Configuration Management

### 7.1 Tunable Parameters (config_settings table)
- `confidence_threshold_auto_store` (float, default: 0.85)
- `confidence_threshold_review` (float, default: 0.60)
- `always_review` (boolean, default: false)
- `cache_expiry_days` (integer, default: 365)
- `max_retry_attempts` (integer, default: 3)
- `llm_default_provider` (string, default: "openai")
- `llm_default_model` (string, default: "gpt-4-turbo-preview")
- `cost_alert_threshold_daily` (float, default: 10.00 USD)
- `enable_auto_matching` (boolean, default: true)
- `match_name_threshold` (float, default: 0.90)

### 7.2 Environment Variables
- `OPENAI_API_KEY` - OpenAI API key
- `GRAMPS_WEB_URL` - Gramps Web instance URL
- `GRAMPS_API_TOKEN` - Gramps Web authentication token
- `MARIADB_HOST` - MariaDB hostname
- `MARIADB_PORT` - MariaDB port (default: 3306)
- `MARIADB_USER` - MariaDB username
- `MARIADB_PASSWORD` - MariaDB password
- `MARIADB_DATABASE` - Database name (default: genealogy_cache)

## 8. Deployment Architecture

### 8.1 Container Configuration

#### Frontend Container
- Base image: Node.js (for React/Vue)
- Exposed port: 3000 (or 8080)
- Volume: None (stateless)
- Environment: Backend API URL

#### Backend Container
- Base image: Python 3.11+
- Exposed port: 5000 (Flask) or 8000 (FastAPI)
- Volumes:
  - Config files
  - Logs directory
- Environment: All API keys, database credentials

#### Gramps Web Container
- Official Gramps Web image
- Exposed port: 5555
- Volumes:
  - Gramps database
  - Media files
- Environment: Gramps configuration

#### MariaDB Container
- Official MariaDB 10.11+ image
- Exposed port: 3306 (internal only)
- Volumes:
  - Database data directory
  - Init scripts (schema.sql)
- Environment: Root password, database name

### 8.2 Podman Pod / Network
- Create dedicated Podman network: `genealogy-net`
- All containers on same network
- Service discovery by container name
- No external access to MariaDB (security)
- Frontend and Backend exposed to host

### 8.3 Container Orchestration
- Use podman-compose.yml for multi-container deployment
- Health checks for all services
- Restart policies: unless-stopped
- Resource limits (CPU, memory)
- Logging configuration (JSON driver)

## 9. Future Phase Considerations

### 9.1 Phase 2: TruePeopleSearch Integration
- Person lookup by name and location
- Address history extraction
- Relative identification
- Phone number and email (if available)
- Cross-reference with Gramps data
- New MariaDB tables for TruePeopleSearch cache
- Confidence scoring for public record matches

### 9.2 Phase 3: 23andMe Integration
- OAuth 2.0 authentication flow
- DNA match retrieval
- Relationship calculation from cM shared
- Haplogroup information
- Merge DNA relationships with documentary evidence
- New tables for genetic data
- Privacy considerations (explicit consent)

### 9.3 Phase 4: Enhanced Features
- Newspaper archive integration (Newspapers.com, Ancestry.com)
- Census record extraction
- Find A Grave integration
- Church record scraping
- Historical document OCR
- Photo facial recognition (identify people in photos)
- Timeline visualization
- Automated consistency checking (conflicting dates, impossible relationships)
- Collaborative features (share findings, merge trees)
- Mobile application (iOS/Android)

### 9.4 Multi-LLM Support
- Abstract LLM interface
- Provider implementations:
  - OpenAI (GPT-4, GPT-3.5)
  - Anthropic Claude
  - Local models (Ollama, LLaMA)
  - Google Gemini
  - Open-source alternatives
- Cost comparison and model selection
- Fallback chain (primary fails → secondary → tertiary)
- A/B testing for accuracy comparison

## 10. Security & Privacy

### 10.1 Data Privacy
- Store only publicly available information
- No authentication required (single-user system initially)
- Data retention policies for cached content
- GDPR considerations for future multi-user
- Provide data export and deletion capabilities
- Respect copyright (store minimal obituary content)

### 10.2 Container Security
- Non-root users in containers
- Read-only root filesystems where possible
- No privileged containers
- Secrets management (podman secrets)
- Regular security updates (base images)
- Vulnerability scanning (Trivy, Clair)

### 10.3 API Security
- API keys stored in environment variables (never in code)
- Secure communication (HTTPS) for external APIs
- Rate limiting to prevent abuse
- Input validation and sanitization
- SQL injection prevention (parameterized queries)
- XSS prevention (frontend sanitization)

### 10.4 Future Authentication
- OAuth 2.0 or JWT tokens
- Role-based access control (RBAC)
- User account management
- Multi-factor authentication (MFA)
- Audit logging of user actions

## 11. Success Metrics

### 11.1 Phase 1 Success Criteria
- Successfully extract and store data from 90%+ of submitted obituary URLs
- Correctly identify primary deceased individual in 95%+ of cases
- Correctly identify relationships with ≥80% accuracy
- Auto-store rate ≥60% (without manual review)
- Average cost per obituary < $0.10
- Cache hit rate ≥70% for repeated processing
- Zero data loss incidents
- GEDCOM validation passes for all stored data
- Processing time < 30 seconds per obituary

### 11.2 Quality Metrics
- User correction rate (% of auto-stored records that need editing)
- False positive match rate (incorrect Gramps record linking)
- False negative match rate (missed opportunities to link)
- Average confidence score for auto-stored records
- Review queue backlog size
- LLM accuracy (precision/recall on test set)

### 11.3 Performance Metrics
- Average processing time per obituary
- Cache hit rate (overall and by cache layer)
- API response times (95th percentile)
- Database query performance
- Concurrent processing capacity
- Container resource utilization

### 11.4 Cost Metrics
- Daily/monthly LLM API costs
- Cost per obituary processed
- Cost savings from caching
- Token usage trends
- Cost per successfully stored person record

## 12. Testing Strategy

### 12.1 Unit Tests
- LLM prompt/response parsing
- Entity extraction logic
- Confidence scoring algorithms
- Matching algorithms (name, date, location)
- Cache key generation
- GEDCOM export validation

### 12.2 Integration Tests
- End-to-end obituary processing
- Gramps Web API interactions
- MariaDB CRUD operations
- LLM API mocking (avoid costs in tests)
- Container startup and communication

### 12.3 Test Data
- Curated set of 50 test obituaries covering:
  - Simple cases (nuclear family)
  - Complex cases (multiple marriages, large families)
  - Edge cases (missing data, ambiguous relationships)
  - International names
  - Various obituary formats
- Known ground truth for accuracy measurement

### 12.4 Validation
- GEDCOM export from Gramps Web
- Import into other genealogy software (Ancestry, FamilySearch)
- Verify data integrity and relationship accuracy

## 13. Documentation Requirements

### 13.1 User Documentation
- Quick start guide
- How to process an obituary
- Understanding confidence scores
- Manual review workflow
- Configuration guide
- Troubleshooting common issues

### 13.2 Technical Documentation
- Architecture overview with diagrams
- Container setup and deployment
- Database schema documentation
- API endpoint documentation
- LLM prompt engineering guide
- Contribution guidelines (for future open-source)

### 13.3 Developer Documentation
- Code structure and organization
- Development environment setup
- Running tests
- Adding new LLM providers
- Adding new data sources
- Database migration procedures

## 14. Known Limitations & Constraints

### 14.1 Technical Constraints
- LLM extraction accuracy depends on obituary quality
- No access to paywalled obituaries
- JavaScript-heavy sites may require additional tools (Playwright)
- Ambiguous relationships difficult to resolve automatically
- Name variations and nicknames challenging to match
- Historical date format variations

### 14.2 Business Constraints
- LLM API costs (mitigated by caching)
- Rate limits on external services
- Copyright restrictions on obituary content
- No guarantee of obituary availability (links may break)

### 14.3 Phase 1 Limitations
- Single obituary at a time (no batch upload)
- English language only
- No image/photo processing
- No automatic source discovery
- Manual data correction required for low-confidence extractions
- No collaborative features

## 15. Open Questions & Decisions

### 15.1 Resolved Decisions
✅ LLM Provider: OpenAI initially, multi-provider in future
✅ Cache Storage: MariaDB
✅ Container Platform: Podman
✅ Backend Language: Python
✅ Always Review: Optional checkbox, configurable auto-store
✅ Matching Strategy: Automatic with confidence thresholds
✅ Multi-Obituary Linking: Yes, link all sources to same person

### 15.2 Remaining Decisions
1. **Frontend Framework**: React, Vue, or Svelte? (Recommendation: React for ecosystem)
2. **Python Web Framework**: Flask or FastAPI? (Recommendation: FastAPI for async and auto-docs)
3. **ORM**: SQLAlchemy, Tortoise, or raw SQL? (Recommendation: SQLAlchemy)
4. **Task Queue**: Celery for async processing? Or simple threading?
5. **LLM Prompt Strategy**: Few-shot examples vs. zero-shot with detailed instructions?
6. **Matching Algorithm**: Exact match, fuzzy matching (fuzzywuzzy), or ML-based?
7. **Container Registry**: Docker Hub, GitHub Container Registry, or local only?
8. **Backup Strategy**: Automated database backups? Frequency?

## 16. Development Phases

### 16.1 Phase 1.1: Foundation (Weeks 1-2)
- Set up Podman containers (MariaDB, Gramps Web)
- Create MariaDB schema
- Basic backend API structure (FastAPI)
- Database connection and ORM setup
- Configuration management

### 16.2 Phase 1.2: Core Processing (Weeks 3-4)
- Web scraping implementation
- OpenAI API integration
- Entity extraction with LLM
- Caching logic (all three layers)
- Basic Gramps Web integration

### 16.3 Phase 1.3: Matching & Storage (Weeks 5-6)
- Matching algorithm implementation
- Confidence scoring logic
- Gramps Web record creation
- Relationship mapping
- Testing with sample obituaries

### 16.4 Phase 1.4: UI & Review (Weeks 7-8)
- Frontend development
- Input page
- Processing page with progress
- Review interface for flagged entities
- Results page
- Configuration page

### 16.5 Phase 1.5: Polish & Testing (Weeks 9-10)
- Cost tracking dashboard
- Error handling improvements
- Performance optimization
- Comprehensive testing
- Documentation
- Bug fixes

## 17. Out of Scope (Phase 1)

- User account management
- Authentication/authorization
- Mobile application
- Automated source discovery
- Image/photo processing
- Social media integration
- Collaborative features
- Advanced visualizations (beyond basic tree)
- Export to other formats (beyond GEDCOM)
- Batch processing of multiple URLs
- Scheduled/automated processing
- Email notifications
- API for external integrations
- Multi-language support
- Historical document OCR
- Census record extraction
- DNA data integration (Phase 3)
- TruePeopleSearch integration (Phase 2)

---

**Document Version**: 2.0  
**Last Updated**: November 25, 2025  
**Status**: Ready for Development  
**Next Review**: Upon completion of Phase 1.2