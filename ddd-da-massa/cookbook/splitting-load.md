# Splitting Load

## Principle

> When cognitive load exceeds limits, split the code. But only split for valid reasons.

## Valid Reasons to Create New Classes

1. **Cognitive load exceeded** — Must distribute
2. **New domain concept** — Entity, Value Object
3. **Language feature usage** — Enum, sealed type, interface

## Invalid Reasons

- "Feels cleaner"
- "Follows some pattern"
- "Separates concerns" (without measurable benefit)

## When to Split

### Signal: Over 7 Points in Service

```java
// ❌ 10+ points - needs splitting
@ApplicationScoped
public class PerformanceReviewService {

    @Inject private GoalRepository goals;           // +1
    @Inject private SystemUserRepository users;     // +1
    @Inject private ReviewRepository reviews;       // +1

    @Transactional
    public void save(NewReviewForm form) {
        List<GoalReview> goalReviews = form.getReviewForms()
            .stream()                               // +1
            .map(rf -> {                            // +1
                Goal goal = goals.findById(rf.goalId()).get();
                return new GoalReview(goal, rf.positivePoints(),
                    rf.improvements(), rf.nextStep());
            })
            .toList();                              // +1

        SystemUser employee = users.findById(form.employeeId()).get();  // +1

        Optional<SalaryInfo> salaryInfo =
            form.isSalaryReview()                   // +1 (ternary)
                ? Optional.of(form.salaryForm().toModel())
                : Optional.empty();

        PerformanceReview review = new PerformanceReview(
            employee, goalReviews, salaryInfo);     // +1

        reviews.save(review);
    }
}
// Total: ~10 points ❌
```

### Solution: Extract to Form Value Object

```java
// Form handles conversion
public record NewReviewForm(
    @NotNull Long employeeId,
    @NotEmpty List<@Valid GoalReviewForm> reviewForms,
    boolean salaryReview,
    @Valid SalaryReviewForm salaryForm
) {
    public PerformanceReview toEntity(
            SystemUser employee,
            GoalRepository goals) {

        List<GoalReview> goalReviews = reviewForms.stream()
            .map(rf -> rf.toEntity(goals))
            .toList();

        Optional<SalaryInfo> salaryInfo = salaryReview
            ? Optional.of(salaryForm.toModel())
            : Optional.empty();

        return new PerformanceReview(employee, goalReviews, salaryInfo);
    }
}

public record GoalReviewForm(
    @NotNull Long goalId,
    String positivePoints,
    String improvements,
    NextStep nextStep
) {
    public GoalReview toEntity(GoalRepository goals) {
        Goal goal = goals.findById(goalId)
            .orElseThrow(GoalNotFoundException::new);
        return new GoalReview(goal, positivePoints, improvements, nextStep);
    }
}

// Service is now simple
@ApplicationScoped
public class PerformanceReviewService {

    @Inject private GoalRepository goals;       // +1
    @Inject private SystemUserRepository users; // +1
    @Inject private ReviewRepository reviews;   // +1

    @Transactional
    public void save(NewReviewForm form) {
        SystemUser employee = users.findById(form.employeeId())
            .orElseThrow(NotFoundException::new);   // +1

        PerformanceReview review = form.toEntity(employee, goals);  // +1
        reviews.save(review);
    }
}
// Total: 5 points ✓
```

## Splitting Strategies

| Strategy             | When to Use                     |
| -------------------- | ------------------------------- |
| Move to Form/DTO     | Complex conversion logic        |
| Move to Entity       | Logic on entity state           |
| Extract helper class | Reusable utility logic          |
| Extract service      | Independent business capability |

## New Class Also Has Limits

The extracted class must also respect limits:

```java
// If Form has too much logic (>9 points), extract further
public record ComplexForm(...) {

    // This helper could be extracted if form exceeds 9 points
    private List<Item> convertItems(Repository repo) {
        // Complex conversion
    }
}
```

## Package-Private for Internal Extraction

```java
// Package-private - not part of public API
@ApplicationScoped
class GoalReviewConverter {

    @Inject
    private GoalRepository goals;

    List<GoalReview> convert(List<GoalReviewForm> forms) {
        return forms.stream()
            .map(this::toEntity)
            .toList();
    }
}
```

## Related Entries

- [load-distribution](load-distribution.md) - When to split
- [form-value-objects](form-value-objects.md) - DTOs with logic
- [cohesive-services](cohesive-services.md) - Service splitting
