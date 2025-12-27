# Replace Subclass with Delegate

## Intent

Replace inheritance with composition (delegation) when subclasses are used for one axis of variation and may need to change at runtime.

## Code Smells That Indicate This Refactoring

- Object's type needs to change over time
- Inheriting for a single aspect of variation
- Multiple inheritances needed (not supported in Java)
- Subclassing creates tight coupling
- Class hierarchy is becoming complex

## Mechanics

1. Create a new class for the delegate
2. Add a field in the superclass to hold the delegate
3. Move subclass methods to the delegate
4. Create delegate in subclass constructor
5. Move subclass methods to call delegate
6. Move creation of delegate to superclass
7. Remove subclasses

## Example

### Before

```java
public class Booking {
    protected Show show;
    protected LocalDate date;

    public Booking(Show show, LocalDate date) {
        this.show = show;
        this.date = date;
    }

    public boolean hasTalkback() {
        return show.hasTalkback() && !isPeakDay();
    }

    public double getBasePrice() {
        return show.getPrice();
    }

    protected boolean isPeakDay() {
        return date.getDayOfWeek() == DayOfWeek.SATURDAY;
    }
}

public class PremiumBooking extends Booking {
    private PremiumExtras extras;

    public PremiumBooking(Show show, LocalDate date, PremiumExtras extras) {
        super(show, date);
        this.extras = extras;
    }

    @Override
    public boolean hasTalkback() {
        return show.hasTalkback();  // Premium always gets talkback
    }

    @Override
    public double getBasePrice() {
        return super.getBasePrice() + extras.getPremiumFee();
    }

    public boolean hasDinner() {
        return extras.hasDinner() && !isPeakDay();
    }
}
```

### After

```java
public class Booking {
    protected Show show;
    protected LocalDate date;
    private PremiumDelegate premiumDelegate;

    public Booking(Show show, LocalDate date) {
        this.show = show;
        this.date = date;
    }

    public static Booking createPremium(Show show, LocalDate date, PremiumExtras extras) {
        Booking booking = new Booking(show, date);
        booking.premiumDelegate = new PremiumDelegate(booking, extras);
        return booking;
    }

    public boolean hasTalkback() {
        return premiumDelegate != null
            ? premiumDelegate.hasTalkback()
            : show.hasTalkback() && !isPeakDay();
    }

    public double getBasePrice() {
        return premiumDelegate != null
            ? premiumDelegate.getBasePrice()
            : show.getPrice();
    }

    public boolean hasDinner() {
        return premiumDelegate != null && premiumDelegate.hasDinner();
    }

    protected boolean isPeakDay() {
        return date.getDayOfWeek() == DayOfWeek.SATURDAY;
    }

    public boolean isPremium() {
        return premiumDelegate != null;
    }
}

public class PremiumDelegate {
    private final Booking host;
    private final PremiumExtras extras;

    public PremiumDelegate(Booking host, PremiumExtras extras) {
        this.host = host;
        this.extras = extras;
    }

    public boolean hasTalkback() {
        return host.show.hasTalkback();
    }

    public double getBasePrice() {
        return host.show.getPrice() + extras.getPremiumFee();
    }

    public boolean hasDinner() {
        return extras.hasDinner() && !host.isPeakDay();
    }
}
```

### With Strategy Interface

```java
public interface BookingType {
    boolean hasTalkback(Booking booking);
    double getBasePrice(Booking booking);
}

public class StandardBookingType implements BookingType {
    @Override
    public boolean hasTalkback(Booking booking) {
        return booking.getShow().hasTalkback() && !booking.isPeakDay();
    }

    @Override
    public double getBasePrice(Booking booking) {
        return booking.getShow().getPrice();
    }
}

public class PremiumBookingType implements BookingType {
    private final PremiumExtras extras;

    public PremiumBookingType(PremiumExtras extras) {
        this.extras = extras;
    }

    @Override
    public boolean hasTalkback(Booking booking) {
        return booking.getShow().hasTalkback();
    }

    @Override
    public double getBasePrice(Booking booking) {
        return booking.getShow().getPrice() + extras.getPremiumFee();
    }
}
```

## When to Use

✅ **Use Replace Subclass with Delegate when:**

- Type needs to change at runtime
- You need multiple axes of variation
- Inheritance is too rigid
- You want looser coupling

❌ **Avoid Replace Subclass with Delegate when:**

- Inheritance is working well
- Type never changes after construction
- The complexity isn't worth it
- Subclasses have very different behavior

## Related Refactorings

- **Replace Superclass with Delegate**: Similar for superclass
- **Replace Type Code with Subclasses**: The inverse
- **Extract Class**: To create the delegate
