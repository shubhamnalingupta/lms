# TechieGigs LMS

A gamified learning platform, first used by TechieGigs and resold white-label to other education businesses.

## Language

**Platform**:
The one product we build and resell; every **Client** runs on it.
_Avoid_: App (ambiguous), system

**Client**:
A business that licenses the **Platform** to teach its own **Learners**. TechieGigs is the first Client.
_Avoid_: Customer, tenant (in conversation), institute

**Client Site**:
One **Client**'s isolated backend instance, holding only that Client's data.
_Avoid_: Tenant DB, workspace

**Branded App**:
A **Client**'s own copy of the learner app, with its name, logo and store listing, available on Android, iOS and web.
_Avoid_: White-label build, flavour

**Learner**:
A person who takes courses on a **Client**'s **Branded App**.
_Avoid_: Student, user, member

**Upstream**:
The open-source Frappe LMS project we build on and pull fixes and updates from.
_Avoid_: Base, core, vendor

**Enrollment**:
A **Learner**'s access to one course on a **Client Site**, granted after they pay outside the app (sales team or web checkout). The **Branded App** only shows courses the Learner has an Enrollment in.
_Avoid_: Purchase, subscription, admission
