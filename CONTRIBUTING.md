# Contributing to MESH Protocol

Thank you for your interest in MESH! Security-focused federation is hard, and we welcome thoughtful contributions.

## Priority: Security Review

**We especially welcome security review.** If you have cryptography or distributed systems expertise, your eyes on [SECURITY.md](SECURITY.md) and [SPECIFICATION.md](SPECIFICATION.md) are invaluable.

### Security Review Focus Areas

1. **Cryptographic choices** — Is Ed25519 + X25519 sufficient? Are there better options?
2. **Trust model** — Are there attacks we haven't considered?
3. **Revocation** — Is the revocation propagation model robust?
4. **Directory security** — Can malicious directories cause harm we haven't mitigated?
5. **Privacy** — What metadata leaks have we missed?

## How to Contribute

### 1. Open an Issue

For significant changes, start with discussion:

- **Bug reports** — Something in the spec doesn't make sense
- **Security concerns** — Potential vulnerabilities (use private reporting for serious issues)
- **Feature proposals** — New capabilities
- **Clarifications** — Something is unclear

### 2. Submit a Pull Request

For concrete improvements:

- **Spec clarifications** — Better explanations, examples
- **Security improvements** — Stronger mitigations
- **Error corrections** — Typos, inconsistencies
- **Additional documentation** — Implementation guides, FAQs

### PR Guidelines

1. **One concern per PR** — Don't bundle unrelated changes
2. **Explain your reasoning** — Why is this change needed?
3. **Consider backwards compatibility** — Breaking changes need strong justification
4. **Update all affected docs** — Changes may touch multiple files

## Specification Changes

The spec is in **draft** status. Now is the time for significant input.

### Change Process

1. Open an issue describing the change
2. Discuss with maintainers and community
3. Draft the change in a PR
4. Review period (minimum 1 week for significant changes)
5. Merge after consensus

### What Requires Discussion First

- New cryptographic primitives
- Changes to wire protocol
- New trust model concepts
- Breaking changes to existing features

### What Can Go Directly to PR

- Typo fixes
- Clarifications that don't change meaning
- Additional examples
- Documentation improvements

## Implementations

Building a MESH implementation? Awesome!

1. Follow the spec closely
2. Run the compliance tests (coming soon)
3. Open an issue if the spec is unclear
4. Add your implementation to the README table

## Code of Conduct

- Be respectful and constructive
- Assume good intent
- Focus on the technical merits
- No personal attacks

Security is serious. Debates about security design can be intense. Keep it professional.

## Security Vulnerabilities

**Do not open public issues for security vulnerabilities.**

See [SECURITY.md](SECURITY.md) for responsible disclosure process.

## License

Contributions are licensed under MIT, same as the project.

---

*Thank you for helping build a more secure knowledge network.*
