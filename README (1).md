import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.*;

// =========================================================
// ENUMS
// =========================================================

enum PermitType {
    RESIDENT, COMMUTER
}

enum VehicleType implements RateModifier {
    CAR {
        public BigDecimal apply(BigDecimal current) {
            return current; // no change
        }
    },
    SUV {
        public BigDecimal apply(BigDecimal current) {
            return current.multiply(BigDecimal.valueOf(1.15));
        }
    },
    MOTORCYCLE {
        public BigDecimal apply(BigDecimal current) {
            return current.multiply(BigDecimal.valueOf(0.70));
        }
    };
}

// =========================================================
// INTERFACES
// =========================================================

interface RateModifier {
    BigDecimal apply(BigDecimal currentMonthly);
}

interface PricingStrategy {
    BigDecimal computeMonthly(PermitSelection sel);
}

interface Validatable {
    void validate();
}

// =========================================================
// EXCEPTION
// =========================================================

class InvalidSelectionException extends RuntimeException {
    public InvalidSelectionException(String msg) {
        super(msg);
    }
}

// =========================================================
// DOMAIN CLASSES
// =========================================================

class CarpoolDiscount implements RateModifier {
    @Override
    public BigDecimal apply(BigDecimal current) {
        return current.multiply(BigDecimal.valueOf(0.90));
    }
}

class PricingPipeline {

    private final List<RateModifier> modifiers;

    public PricingPipeline(List<RateModifier> modifiers) {
        this.modifiers = modifiers;
    }

    public BigDecimal applyAll(BigDecimal start) {
        BigDecimal result = start;
        for (RateModifier m : modifiers) {
            result = m.apply(result);
        }
        return result;
    }
}

class ResidentPricingStrategy implements PricingStrategy {
    @Override
    public BigDecimal computeMonthly(PermitSelection sel) {
        return BigDecimal.valueOf(45.00);
    }
}

class CommuterPricingStrategy implements PricingStrategy {
    @Override
    public BigDecimal computeMonthly(PermitSelection sel) {
        BigDecimal base = BigDecimal.valueOf(35.00);
        return base.multiply(BigDecimal.valueOf(0.85)); // auto 15% off
    }
}

final class PermitSelection implements Validatable {

    private final PermitType permitType;
    private final VehicleType vehicleType;
    private final boolean carpool;
    private final int months;

    public PermitSelection(PermitType permitType,
                           VehicleType vehicleType,
                           boolean carpool,
                           int months) {
        this.permitType = permitType;
        this.vehicleType = vehicleType;
        this.carpool = carpool;
        this.months = months;
        validate();
    }

    public PermitType getPermitType() { return permitType; }
    public VehicleType getVehicleType() { return vehicleType; }
    public boolean isCarpool() { return carpool; }
    public int getMonths() { return months; }

    @Override
    public void validate() {
        if (months < 1 || months > 12)
            throw new InvalidSelectionException("Months must be 1–12.");
        if (permitType == null || vehicleType == null)
            throw new InvalidSelectionException("Permit type and vehicle type are required.");
    }
}

final class PricingCalculator {

    private final PricingStrategy strategy;
    private final PricingPipeline pipeline;

    public PricingCalculator(PricingStrategy strategy, PricingPipeline pipeline) {
        this.strategy = strategy;
        this.pipeline = pipeline;
    }

    public BigDecimal computeSubtotal(PermitSelection sel) {
        BigDecimal monthly = strategy.computeMonthly(sel);
        monthly = pipeline.applyAll(monthly);
        return monthly.multiply(BigDecimal.valueOf(sel.getMonths()));
    }

    public BigDecimal computeCampusFee(BigDecimal subtotal) {
        return subtotal.multiply(BigDecimal.valueOf(0.05));
    }

    public BigDecimal computeTotal(BigDecimal subtotal) {
        return subtotal.add(computeCampusFee(subtotal));
    }
}

final class Receipt {

    public static String format(
            PermitSelection sel,
            BigDecimal subtotal,
            BigDecimal fee,
            BigDecimal total
    ) {
        return String.format("""
                ============ PARKING PERMIT RECEIPT ============
                Permit Type:      %s
                Vehicle Type:     %s
                Carpool:          %s
                Months:           %d

                Subtotal:         $%.2f
                Campus Fee (5%%): $%.2f
                TOTAL:            $%.2f
                ================================================
                """,
                sel.getPermitType(),
                sel.getVehicleType(),
                sel.isCarpool() ? "YES" : "NO",
                sel.getMonths(),
                subtotal.setScale(2, RoundingMode.HALF_UP),
                fee.setScale(2, RoundingMode.HALF_UP),
                total.setScale(2, RoundingMode.HALF_UP)
        );
    }
}

// =========================================================
// MAIN (UI ONLY)
// =========================================================

public class Main {

    public static void main(String[] args) {

        Scanner sc = new Scanner(System.in);

        while (true) {
            try {
                System.out.print("Enter permit type (RESIDENT/COMMUTER): ");
                PermitType permit =
                        PermitType.valueOf(sc.nextLine().trim().toUpperCase());

                System.out.print("Enter vehicle type (CAR/SUV/MOTORCYCLE): ");
                VehicleType vehicle =
                        VehicleType.valueOf(sc.nextLine().trim().toUpperCase());

                System.out.print("Carpool? (Y/N): ");
                boolean carpool =
                        sc.nextLine().trim().equalsIgnoreCase("Y");

                System.out.print("Months (1–12): ");
                int months =
                        Integer.parseInt(sc.nextLine().trim());

                // Build selection (validates internally)
                PermitSelection sel =
                        new PermitSelection(permit, vehicle, carpool, months);

                // Polymorphism: choose strategy dynamically
                PricingStrategy strategy =
                        (permit == PermitType.RESIDENT)
                                ? new ResidentPricingStrategy()
                                : new CommuterPricingStrategy();

                // Pipeline: vehicle modifier + optional carpool
                PricingPipeline pipeline = carpool
                        ? new PricingPipeline(List.of(vehicle, new CarpoolDiscount()))
                        : new PricingPipeline(List.of(vehicle));

                // Calculator
                PricingCalculator calc = new PricingCalculator(strategy, pipeline);

                BigDecimal subtotal = calc.computeSubtotal(sel);
                BigDecimal fee = calc.computeCampusFee(subtotal);
                BigDecimal total = calc.computeTotal(subtotal);

                System.out.println(Receipt.format(sel, subtotal, fee, total));

            } catch (Exception e) {
                System.out.println("\nERROR: " + e.getMessage());
                System.out.println("Please try again.\n");
            }
        }
    }
}
