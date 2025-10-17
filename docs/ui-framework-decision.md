# VaultLink UI Framework Decision

**Document Version:** 1.0  
**Date:** October 17, 2025  
**Client:** Synertec  
**Project:** VaultLink MVP AWS Rebuild  
**Purpose:** UI Library vs Custom Framework Analysis

---

## Executive Summary

This document analyzes the trade-offs between adopting an established UI component library versus building a custom UI framework for VaultLink MVP. Given project constraints (20-week timeline, "nicer looking" goal, mobile-first, white-label branding), this analysis provides a clear recommendation.

**TL;DR Recommendation:** **Use Angular Material with custom theming** for MVP, plan custom components for Phase 2 if needed.

---

## Context & Requirements

### VaultLink UI Requirements

**Functional:**
- Mobile-first responsive design (3G performance target)
- White-label branding (logos, colors, fonts per customer)
- Statement viewing with aging breakdown visualization
- Payment flow (Stripe Elements integration)
- Query submission forms
- Admin configuration portal
- QR code scanning interface

**Non-Functional:**
- "Nicer looking than current desktop app" (key stakeholder goal)
- Accessibility WCAG 2.1 AA compliance
- Performance: <3s page load on 3G mobile
- Consistent UX across all customer brands

**Constraints:**
- 20-week MVP timeline
- Small team (3-6 developers)
- Budget-conscious (consulting engagement)

---

## Option 1: Established UI Library

### Libraries Considered

#### 1. Angular Material (Recommended)
- **Description:** Official Material Design component library for Angular
- **Maturity:** Stable, v20+ aligned with Angular 20+
- **Components:** 40+ components (forms, navigation, data tables, dialogs)
- **Theming:** Built-in theming system with CSS custom properties
- **Documentation:** https://material.angular.dev/components/categories

#### 2. PrimeNG
- **Description:** Rich UI component suite with 90+ components
- **Maturity:** 10+ years, enterprise-focused
- **Components:** Advanced data tables, charts, file upload
- **Theming:** Multiple pre-built themes, SCSS customization
- **Documentation:** https://primeng.org/autocomplete

#### 3. Ant Design (NG-ZORRO)
- **Description:** Angular implementation of Ant Design
- **Maturity:** Stable, backed by Alibaba
- **Components:** 60+ components with excellent mobile support
- **Theming:** Less customization system
- **Documentation:** https://ng.ant.design/components/overview/en

#### 4. Bootstrap + ng-bootstrap
- **Description:** Bootstrap 5 with Angular directives
- **Maturity:** Industry standard, widely known
- **Components:** 20+ components, requires more custom work
- **Theming:** SCSS variables, extensive customization
- **Documentation:** https://ng-bootstrap.github.io/#/components/accordion/overview

---

### Pros: Using UI Library

####  Speed to Market
- Pre-built components ready to use immediately
- Significantly faster development than building from scratch
- Allows team to focus on VaultLink's unique business features

####  Quality & Reliability
- Battle-tested by thousands of production applications
- Cross-browser and mobile compatibility guaranteed
- Community actively maintains and improves the library

####  Accessibility Built-In
- WCAG 2.1 AA compliance out of the box
- Screen reader and keyboard navigation support included
- Meets legal and regulatory accessibility requirements

####  White-Label Theming
- Easy per-customer branding (colors, logos, fonts)
- Runtime theme switching without system rebuild
- Consistent branding across all UI components

---

### Cons: Using UI Library

####  Design Limitations
- Material Design has a recognizable look that may not feel unique
- Limited to components provided by the library

**Mitigation:** Custom theming and mixing in custom-built components where differentiation is needed

####  Slightly Larger File Size
- UI libraries add to the initial download size
- May impact 3G mobile load times slightly

**Mitigation:** Optimize with lazy loading and code splitting techniques

####  Third-Party Dependency
- Reliant on library maintainers for updates and bug fixes

**Mitigation:** Choose established, well-maintained libraries backed by major organizations

---

## Option 2: Custom UI Framework

### What "Custom" Means

Building reusable Angular components from scratch:
- Custom button, input, select, checkbox components
- Custom layout system (grid, flex utilities)
- Custom navigation (header, sidebar, breadcrumbs)
- Custom data display (cards, tables, lists)
- Custom overlays (modals, toasts, tooltips)

---

### Pros: Custom UI Framework

####  Complete Design Control
- Pixel-perfect match to any design vision
- Truly unique, differentiated look and feel
- UI becomes a competitive differentiator

####  Optimized File Size
- Only include code that's actually needed
- Potentially faster load times on mobile devices

####  Full Flexibility
- Complete control over component behavior
- No workarounds needed for custom requirements

####  No Third-Party Dependencies
- Full ownership of the code
- No reliance on external maintainers

---

### Cons: Custom UI Framework

####  Significantly Longer Development Time
- Building a complete component library takes considerably longer
- Ongoing maintenance and bug fixing required
- Diverts team focus from VaultLink's core business features

####  Quality & Testing Burden
- Must test extensively across all browsers and devices
- Full responsibility for all defects and issues
- Higher initial defect rate compared to mature libraries

####  Accessibility Compliance
- Must implement WCAG 2.1 AA compliance from scratch
- Complex to implement properly
- Risk of failing accessibility requirements

####  Documentation & Training
- Must create and maintain internal documentation
- Slower onboarding for new team members
- Knowledge concentrated within team

####  Resource Investment
- Team has capability but requires significant time investment
- Time spent on UI infrastructure rather than business value

---

## Hybrid Approach: Library + Custom Components

### Strategy

**Use Angular Material as foundation, build custom components where needed.**

**Library Components (80%):**
- Forms (inputs, selects, checkboxes, radio buttons)
- Buttons and button groups
- Navigation (toolbar, sidenav, tabs)
- Layout (grid, cards, dividers)
- Overlays (dialogs, snackbars, tooltips)
- Data tables
- Date/time pickers

**Custom Components (20%):**
- Statement aging breakdown visualization (unique to VaultLink)
- QR code scanner interface
- Payment amount selector (with aging bucket targeting)
- Query reason selector (custom dropdown)
- Customer branding preview
- Document viewer with PDF integration

### Benefits

**Best of Both Worlds**
- Fast development using proven library components
- Custom components for VaultLink's unique features
- Professional foundation with brand differentiation

**Achievable Timeline**
- Meets 20-week MVP deadline
- Custom components built incrementally as needed

**Strategic Resource Allocation**
- Team focuses on VaultLink's unique business value
- Leverage existing solutions for common UI patterns

---

## Summary Comparison

| Factor | UI Library (Angular Material) | Custom Framework |
|--------|-------------------------------|------------------|
| **Development Speed** | Fast - components ready to use | Slow - build everything from scratch |
| **Timeline Fit** | Meets 20-week MVP deadline | Extends timeline significantly |
| **Accessibility** | WCAG 2.1 AA built-in | Must implement from scratch |
| **White-Label Theming** | Robust theming system | Full control but more work |
| **Maintenance** | Community maintains | Team maintains everything |
| **Focus** | Business features | UI infrastructure |

---

## Recommendation: Angular Material + Custom Components

### Decision

**Use Angular Material as the primary UI framework with custom components for unique VaultLink features.**

### Rationale

1. **Timeline efficiency:** Using a mature library allows the team to focus on VaultLink's unique business features rather than building UI infrastructure
2. **Quality assurance:** Accessibility and cross-browser support are non-negotiable and come built-in
3. **White-label theming:** Angular Material's theming system solves multi-tenant branding requirements
4. **Mobile performance:** With lazy loading and tree-shaking, bundle size is acceptable
5. **Risk mitigation:** Established library reduces unknowns and blockers

### Implementation Plan

**Phase 1: MVP**
- Install Angular Material library
- Create base theme with Synertec branding
- Build application using Material components
- Develop custom components for VaultLink-specific features
- Apply heavy theming for unique look

**Phase 2: Post-MVP**
- Gather user feedback on interface
- Identify areas needing further customization
- Incrementally replace or enhance components as needed
- Build reusable component library for future projects

---

## White-Label Theming Approach

Angular Material provides a robust theming system that allows:
- Runtime theme switching per customer
- Custom brand colors, logos, and fonts
- Consistent branding across all UI components
- Theme configuration stored per customer in the database

---

## Conclusion

**Recommendation:** Use Angular Material with custom theming and selective custom components.

### Why This Works for VaultLink

1. **Focus on business value:** Team can concentrate on VaultLink's unique features rather than UI infrastructure
2. **Timeline efficiency:** Fits within 20-week MVP timeline
3. **Quality:** Accessibility and cross-browser support guaranteed out of the box
4. **Theming:** White-label multi-tenant branding achievable with built-in theming system
5. **Risk:** Low risk, proven approach with extensive community support
6. **Flexibility:** Can gradually replace or customize components post-MVP if needed

### Next Steps

1. Install Angular Material library and configure base theme
2. Create Synertec brand theme and test multi-tenant capabilities
3. Build core application structure using Material components
4. Develop custom components for VaultLink-specific features
5. Testing and refinement
6. User acceptance testing with customer branding
7. Final polish and launch preparation

---

**For MVP scope, see `brief.md`**  
**For infrastructure details, see `aws-infrastructure-services.md`**  
**For testing approach, see `testing-strategy.md`**  
**For epic breakdown, see `epics.md`**  
**For branching strategy, see `branching-strategy.md`**

---

**Document End**
