---
name: module-security-audit
description: Run a comprehensive security audit checklist against a PrestaShop 9 module covering SQL injection, XSS, CSRF, file upload vulnerabilities, insecure deserialization, permission checks, and rate limiting on sensitive endpoints. Use before publishing a module to the marketplace or deploying to production.
---

## Requirements

Ask the user:
* Module technical name and root path (e.g. `modules/mymodule`).
* PHP namespace (e.g. `MyVendor\Mymodule`).
* Whether the module has: front controllers, admin controllers, AJAX endpoints, file upload handling, payment processing, or webservice resources.
* PrestaShop version target (9.x assumed; flag legacy patterns if found).
* Whether to run automated static analysis (PHPStan with security rules, Psalm taint analysis) in addition to the manual checklist.

## Steps

1. Install static analysis tools for security scanning:

    ```bash
    composer require --dev phpstan/phpstan phpstan/phpstan-strict-rules vimeo/psalm
    ```

    Create a `phpstan-security.neon` focusing on taint tracking:

    ```neon
    includes:
        - vendor/phpstan/phpstan-strict-rules/rules.neon
    parameters:
        level: 8
        paths:
            - src/
            - controllers/
        checkMissingIterableValueType: false
    ```

2. **SQL Injection audit.** Search for raw SQL and verify parameterization:

    * Grep for `Db::getInstance()->execute(`, `Db::getInstance()->getRow(`, `Db::getInstance()->executeS(` and ensure all user input passes through `pSQL()` or `(int)` casting.
    * For Doctrine: verify all QueryBuilder usage binds parameters (`->setParameter(...)`) rather than concatenating values.
    * For CQRS handlers: confirm that IDs and filter values arriving from commands/queries are typed (value objects with validation in the constructor).
    * Flag any usage of `$_GET`, `$_POST`, `$_REQUEST` that flows into SQL without sanitization.

3. **Cross-Site Scripting (XSS) audit:**

    * In Smarty templates: verify all variables use `{$var|escape:'html':'UTF-8'}` or `{$var nofilter}` is justified (rendering trusted HTML only).
    * In Twig templates: confirm autoescape is enabled (default in PS 9). Flag any `|raw` filter usage and verify the source is trusted.
    * In controllers returning JSON: verify `Content-Type: application/json` is set and no user input is reflected in error messages without encoding.
    * Check that `Tools::getValue()` output is never echoed without escaping.

4. **CSRF protection audit:**

    * Admin controllers: verify every state-changing action checks the CSRF token via `$this->isTokenValid()` or Symfony's built-in form CSRF protection.
    * Front controllers: verify forms include `{csrf_token}` and the controller validates it.
    * AJAX endpoints: confirm they either use the PS token system or are protected by the Symfony CSRF token manager.

5. **File upload vulnerability audit:**

    * Check that uploaded files are validated: MIME type check (not just extension), file size limit, filename sanitization.
    * Verify uploads are stored outside the webroot or in a directory with `.htaccess` / nginx rules denying script execution.
    * Flag any `move_uploaded_file()` that uses the original filename without sanitization.
    * Verify image uploads are reprocessed (e.g. via GD or Imagick) to strip embedded payloads.

6. **Authentication and authorization audit:**

    * Admin controllers: verify they check `$this->access('view')`, `$this->access('edit')`, `$this->access('delete')` or use Symfony security voters.
    * Admin API resources: verify OAuth2 scopes are declared and enforced via `#[ApiResource]` security attributes.
    * Front controllers: verify customer authentication checks where required (`$this->context->customer->isLogged()`).
    * Verify no sensitive data (passwords, tokens, keys) is logged or exposed in error responses.

7. **Insecure deserialization audit:**

    * Flag any usage of `unserialize()` with user-controlled input. Require `['allowed_classes' => [...]]` or prefer `json_decode()`.
    * Check cookie/session data for serialized objects that could be tampered with.

8. **Rate limiting and brute force protection:**

    * Verify login-related endpoints (if any) implement rate limiting or account lockout.
    * Check that payment callbacks validate signatures and are idempotent.
    * Verify API endpoints have sensible request size limits.

9. **Sensitive data exposure audit:**

    * Verify no API keys, secrets, or credentials are hardcoded in source files.
    * Check that `.gitignore` excludes config files containing secrets.
    * Verify error pages do not expose stack traces, SQL queries, or internal paths in production mode.
    * Check that `composer.json` does not include `require-dev` packages in production builds.

10. **Dependency vulnerability scan:**

    ```bash
    composer audit
    npm audit  # if the module ships JS
    ```

    Flag any known CVE in direct or transitive dependencies.

11. Generate a security report documenting:
    * Each finding with severity (Critical / High / Medium / Low / Info).
    * The file and line number where the vulnerability exists.
    * A recommended fix for each finding.
    * A pass/fail summary against the PrestaShop Marketplace security requirements.

12. Verify:
    * PHPStan at level 8 passes with no errors.
    * `composer audit` returns zero known vulnerabilities.
    * All SQL queries use parameterized statements.
    * All user-facing output is properly escaped.
    * All state-changing endpoints validate CSRF tokens.
    * All file uploads are validated and sanitized.
    * All admin endpoints enforce proper authorization.

## Do

- Run the audit against the full source tree, not just `src/`. Check `controllers/`, `views/`, `upgrade/`, and root files.
- Treat `Tools::getValue()` as untrusted user input everywhere it appears.
- Verify that `pSQL()` is used for string interpolation in legacy SQL, but prefer Doctrine parameterized queries in new code.
- Check both the happy path and error paths for information leakage.
- Re-run the audit after every fix to ensure no regressions.

## Don't

- Don't rely solely on automated tools. Static analysis misses business logic flaws, broken access control, and context-dependent vulnerabilities.
- Don't assume Symfony's form component handles all CSRF. AJAX endpoints and custom forms need explicit token validation.
- Don't treat `(int)` casting as sufficient for all injection vectors. It only works for integer parameters.
- Don't skip the dependency audit. A single vulnerable transitive dependency can compromise the entire module.
- Don't store security findings in public issue trackers. Report them privately following PrestaShop's responsible disclosure policy.

## Canonical examples

- [devdocs - Security best practices](https://devdocs.prestashop-project.org/9/modules/creation/good-practices/) - the official security guidance for module developers.
- [PrestaShop Security Advisories](https://github.com/PrestaShop/PrestaShop/security/advisories) - real-world CVEs and their fixes.
- [OWASP Top 10](https://owasp.org/www-project-top-ten/) - the industry-standard web application security risks.
- [PHPStan](https://phpstan.org/) - static analysis for PHP.
- [Psalm Security Analysis](https://psalm.dev/docs/security_analysis/) - taint analysis for tracking user input to sensitive sinks.

## Related skills

- `module-add-unit-tests` - unit tests can encode security invariants (e.g. value object validation rejects dangerous input).
- `module-add-integration-tests` - integration tests verify authorization is enforced end-to-end.
- `module-add-api-tests` - API tests verify OAuth2 scopes and error responses.
- `module-validate` - the structural validator catches some security issues (missing index.php, exposed vendor/).
