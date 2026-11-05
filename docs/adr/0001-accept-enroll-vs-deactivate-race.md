# ADR 0001: Accept the race between enrolling and deactivating

**Status:** accepted

**Context.** An inactive patient or medication must not have open enrollments. `EnrollmentService.enroll` checks that both are active, and deactivation closes open enrollments in a listener in the same transaction. If the two run concurrently, the listener may not see the uncommitted enrollment, while `enroll` has already read the patient as active. The result is an open enrollment for an inactive patient or medication. `@Version` does not help because `enroll` only reads the patient, and the foreign key does not help because its `FOR KEY SHARE` lock does not conflict with updating `active`.

**Decision.** Do not lock for now. The window is milliseconds, deactivation is rare, and the result is visible and can be corrected by cancelling the enrollment. Locking would touch two services, two repositories and their tests for a case that is unlikely to occur.

**Consequences.** The invariant can be violated in this one case. If it must always hold, take `PESSIMISTIC_READ` on the patient and medication in `enroll` and `PESSIMISTIC_WRITE` in `deactivate`, and cover it with a concurrent test. The shared lock lets enrollments for the same medication run in parallel.
