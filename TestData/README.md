# Test Data for Content Moderator

This directory contains test case data for validating the Content Moderator Backend API.

## Files

### `technical_tests.json`
Contains **9 test cases** for technical validation:
- **8 negative cases** - Testing request validation (missing fields, invalid values, size limits)
- **1 positive case** - Valid request that should pass validation

**Expected Results:**
- `400` - Bad request (validation error)
- `200` - Request is technically valid

### `business_tests.json`
Contains **20 test cases** for business validation (policy compliance):
- **10 positive cases** - Messages that comply with community policies
- **10 negative cases** - Messages that violate policies (2 of each violation type)

**Expected Results:**
- `positive` - Message complies with policies
- `negative` - Message violates policies

**Violation Types Covered:**
- `HARASSMENT` - Bullying, insults, persistent unwanted contact
- `SPAM` - Commercial advertisements, affiliate links
- `SELF_PROMOTING` - Unsolicited self-promotion, portfolio spam
- `DOXXING` - Sharing personal information without consent
- `IP_DISPUTES` - IP infringement accusations as harassment
- `OFF_TOPIC` - Political or religious debates unrelated to hobby

## Test Case Structure

Each test case follows this format:

```json
{
  "id": "unique-test-id",
  "goal": "Description of what this test validates",
  "message": {
    "id": "numeric-message-id",
    "title": "Message title",
    "body": ["message text", "optional replies"]
  },
  "expected-result": "positive|negative|400|200",
  "expected-reason": "HARASSMENT|SPAM|..." // Optional, for negative cases
}
```

## Usage

### With Test Framework
```bash
# Run all technical tests
python tests/run_tests.py --type technical

# Run all business tests
python tests/run_tests.py --type business

# Run all tests
python tests/run_tests.py --type all
```

### Manual Testing with curl
```bash
# Test a positive case
curl -X POST http://localhost:8000/api/validateMessage \
  -H "Content-Type: application/json" \
  -d '{
    "id": "1001",
    "title": "My first Space Marine painting",
    "body": ["I just finished painting my first Space Marine miniature"]
  }'

# Test a negative case (harassment)
curl -X POST http://localhost:8000/api/validateMessage \
  -H "Content-Type: application/json" \
  -d '{
    "id": "2001",
    "title": "Re: Your terrible paint job",
    "body": ["Your painting is absolute garbage", "Just quit this hobby"]
  }'

# Test technical validation (missing field)
curl -X POST http://localhost:8000/api/validateMessage \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Valid title",
    "body": ["Valid message"]
  }'
```

## Test Coverage Summary

### Technical Validation (9 tests)
| Test ID | Validation Rule | Expected |
|---------|----------------|----------|
| tech-001 | Missing field: id | 400 |
| tech-002 | Missing field: title | 400 |
| tech-003 | Missing field: body | 400 |
| tech-004 | Non-numeric id | 400 |
| tech-005 | Title > 100 chars | 400 |
| tech-006 | Empty body array | 400 |
| tech-007 | Body element > 256 chars | 400 |
| tech-008 | Total body > 10000 chars | 400 |
| tech-009 | Valid request | 200 |

### Business Validation (20 tests)
| Test ID Range | Category | Count | Expected |
|---------------|----------|-------|----------|
| biz-positive-001 to 010 | Compliant messages | 10 | positive |
| biz-negative-001 to 010 | Policy violations | 10 | negative |

**Negative Test Breakdown:**
- 2x HARASSMENT
- 2x SPAM
- 2x SELF_PROMOTING
- 1x DOXXING
- 1x IP_DISPUTES
- 2x OFF_TOPIC

## Expected Test Results

When testing against a correctly implemented Content Moderator API:

- **Technical Tests**: 8/9 should return HTTP 400, 1/9 should return HTTP 200
- **Business Tests**: Depends on LLM evaluation, but should achieve >90% accuracy
  - 10/10 positive cases should return `{"verification": "positive", "reason": "OK"}`
  - 10/10 negative cases should return `{"verification": "negative", "reason": "..."}` with appropriate reason codes

## Notes

- Test cases are designed to be realistic examples from a tabletop miniature hobbyist community
- Business test results may vary slightly due to LLM interpretation
- Some edge cases may be interpreted differently by different LLM models
- Test data aligns with policies defined in `Specification/policies.md`
