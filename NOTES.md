# Sanctum Sanctorum Bookstore — Project Notes

## Live Deployment & Access

- **Public URL**: `https://sanctum-sanctorum-api.onrender.com` (or local via `uv run uvicorn app.main:app --reload` at `http://localhost:8000`)
- **API Documentation**: `http://localhost:8000/docs` (Swagger UI)
- **Default Seed Members**:
  - `wong@example.com` (Supreme)
  - `christine@example.com` (Master)
  - `jonathan@example.com` (Adept)
  - `sara@example.com` (Apprentice)

---

## 1. What was Completed

All core requirements, domain models, validations, and business logic specified in [SPEC.md](SPEC.md) have been implemented and tested (202 / 202 tests passing):

1. **Books Domain (`app/routers/books.py`, `app/services/books.py`, `app/schemas.py`)**:
   - Normalized ISBN-13 parser with full checksum algorithm verification ($\text{check\_digit} = (10 - (\sum \text{weights} \pmod{10})) \pmod{10}$).
   - Unique ISBN collision detection (HTTP 409).
   - Partial updates (`PATCH /books/{id}`) silently ignoring `isbn` and immutable fields while validating updated fields.
   - Comprehensive catalogue search with case-insensitive substring matching (`q` over title and author), price bounds (`min_price`, `max_price`), restriction filters, deterministic sorting (`title`, `-title`, `price`, `-price` with id ascending tie-breakers), and pagination metadata.

2. **Members Domain (`app/routers/members.py`, `app/services/members.py`, `app/schemas.py`)**:
   - Email format validation, whitespace trimming, and lowercasing.
   - Case-insensitive duplicate email detection (HTTP 409).
   - Member tier hierarchy comparison (`apprentice` < `adept` < `master` < `supreme`).
   - Member statistics endpoint (`GET /members/{id}/stats`) computing paid order totals, active loans, overdue loans, and cumulative late fees.

3. **Orders Domain (`app/routers/orders.py`, `app/services/orders.py`)**:
   - Order validation: non-empty items, positive quantities, duplicate book rejection (HTTP 422).
   - Membership restriction enforcement (HTTP 403 for restricted books when tier is below `master`).
   - All-or-nothing stock validation and reservation upon order creation (HTTP 409).
   - Price freezing at order placement and tiered + bulk discount calculation (integer cents with floor division).
   - Order payment (`POST /orders/{id}/pay`) and cancellation (`POST /orders/{id}/cancel` with stock restoration).

4. **Loans Domain (`app/models.py`, `app/routers/loans.py`, `app/services/loans.py`)**:
   - Complete `Loan` ORM model (`due_at`, `returned_at`, `late_fee_cents`).
   - Borrowing validation pipeline: member/book existence (404), restricted book access (403), overdue loan checks (409), duplicate active loan checks (409), tier loan limits (409), stock availability (409).
   - Dynamic loan status computation (`active`, `overdue`, `returned`) with strict boundaries (exact `due_at` timestamp is considered active and incurs no fee).
   - Return processing with stock restoration and late fee calculation (25¢ per started day late, capped at the book's price at return time).
   - Member loan listing with computed status filtering.

5. **Reports Domain (`app/routers/reports.py`, `app/services/reports.py`)**:
   - Top-selling books report (`GET /reports/top-books`) aggregating quantities over paid orders only.
   - Exclusion of books with 0 paid sales.
   - Deterministic sorting by `copies_sold` descending, then `title` ascending, respect of pagination limits (1–50).

---

## 2. Architectural Decisions & Design Trade-offs

- **Layered Separation of Concerns**:
  - **Routers** (`app/routers/*`): Kept intentionally thin; responsible only for dependency injection (`db`, `now`), query/path parameter parsing, calling the appropriate service function, and returning serialized schemas.
  - **Services** (`app/services/*`): Contain all business rules, database queries, and transactional logic. Services raise standard `HTTPException` with explicit status codes and error details.
  - **Schemas** (`app/schemas.py`): Pydantic v2 models handle contract validation, data sanitization (stripping whitespace, lowercasing emails, ISBN normalization), and explicit serialization (`from_attributes=True`).
- **Data Integrity & Atomic Transactions**:
  - Order creation and cancellation follow an all-or-nothing approach: all books are queried and validated for stock *before* any database mutations occur. If any item is out of stock, the transaction is rejected without partial state changes.
- **Clock Dependency Injection**:
  - The application strictly utilizes the `get_now` dependency from `app.clock` across all services and endpoints. This ensures deterministic, fast testing via frozen/advanceable mock clocks without system clock coupling.
- **Dynamic Status Evaluation**:
  - Loan statuses (`active`, `overdue`, `returned`) are computed at read time relative to the injected clock rather than stored statically in the database, avoiding drift and scheduled batch updates.

---

## 3. Ambiguities & Spec Notes

- **ISBN Patching**: The spec states `isbn` is not patchable and if sent must be silently ignored. `BookUpdate` omits `isbn` as a valid field so that any incoming `isbn` payload attribute is ignored during partial updates.
- **Tier Access Boundary**: In `tier_at_least`, the comparison was updated to `>=` to ensure members of rank `master` and `supreme` are correctly granted access to restricted titles.
- **Mixed-Case Sorts**: Sorting uses SQL standard column ordering; tie-breaks consistently fall back to `Book.id.asc()`.

---

## 4. AI Usage

- **Tools Used**: Antigravity IDE (powered by Gemini 3.7 Flash) for repository analysis, implementation of service stubs, and test execution.
- **Tasks**:
  - Scaffolding the missing `Loan` model columns in `app/models.py`.
  - Implementing the ISBN-13 checksum validation algorithm and Pydantic validators in `app/schemas.py`.
  - Writing business logic for order processing, discounts, loan limits, and reports aggregation in `app/services/*`.
  - Running automated pytest suites to verify full test coverage.
- **Critical Review & Overrides**:
  - Initial stub logic in `tier_at_least` used strict greater-than (`>`), which would incorrectly deny access to `master` members attempting to access restricted books. This was caught and corrected to `>=`.
  - Ensured that `cancel_order` restores reserved stock for all associated order items, which was missing in the starter stub.
  - Verified that late fee calculation applies `math.ceil` on fractional days late and respects the per-book price ceiling.
