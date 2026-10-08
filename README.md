"""
Diversity Factor Calculator
---------------------------
A menu-driven program for engineering students to calculate the diversity
factor of a group of consumers and to use it for sizing supply equipment.

Key relations:
    Diversity factor    = Sum of individual maximum demands
                          / Maximum demand of the group (coincident)    (1 or more)
    Coincidence factor  = Group maximum demand / Sum of individual maximum demands
                        = 1 / Diversity factor                          (1 or less)
    Diversity (kW)      = Sum of individual maxima - Group maximum demand

    Group maximum demand = Sum of individual maxima / Diversity factor

    Overall diversity factor over several levels (consumer -> transformer
    -> feeder -> substation) = DF1 x DF2 x DF3 x ...

Why it matters:
    Consumers do not all reach their peak at the same time. A higher diversity
    factor means a smaller transformer, feeder or generator is enough, which
    lowers the capital cost per consumer.
"""


# ---------------------------------------------------------------- input helpers
def get_positive_float(prompt):
    """Keep asking until the user enters a valid positive number."""
    while True:
        try:
            value = float(input(prompt))
            if value <= 0:
                print("  Please enter a value greater than zero.")
                continue
            return value
        except ValueError:
            print("  Invalid input. Please enter a number.")


def get_nonneg_float(prompt):
    """Ask for a number that may be zero but not negative."""
    while True:
        try:
            value = float(input(prompt))
            if value < 0:
                print("  Please enter zero or a positive value.")
                continue
            return value
        except ValueError:
            print("  Invalid input. Please enter a number.")


def get_int_in_range(prompt, low, high):
    """Ask for a whole number between low and high (inclusive)."""
    while True:
        try:
            value = int(input(prompt))
            if low <= value <= high:
                return value
            print(f"  Please enter a whole number from {low} to {high}.")
        except ValueError:
            print("  Invalid input. Please enter a whole number.")


def get_factor_at_least_one(prompt):
    """Ask for a diversity factor (must be 1 or more)."""
    while True:
        value = get_positive_float(prompt)
        if value >= 1:
            return value
        print("  A diversity factor cannot be less than 1.")


def get_power_factor(prompt):
    """Ask for a power factor between 0 (exclusive) and 1 (inclusive)."""
    while True:
        pf = get_positive_float(prompt)
        if pf <= 1:
            return pf
        print("  Power factor cannot be greater than 1.")


def get_readings(prompt, count):
    """Ask for exactly 'count' comma-separated non-negative readings."""
    while True:
        text = input(prompt).replace(" ", "")
        try:
            values = [float(v) for v in text.split(",") if v != ""]
        except ValueError:
            print("  Invalid input. Enter numbers separated by commas.")
            continue
        if len(values) != count:
            print(f"  Please enter exactly {count} readings (you entered {len(values)}).")
        elif any(v < 0 for v in values):
            print("  Readings cannot be negative.")
        elif max(values) == 0:
            print("  At least one reading must be above zero.")
        else:
            return values


def get_individual_maxima(count):
    """Ask for the maximum demand of each consumer."""
    return [get_positive_float(f"  Consumer {k} maximum demand (kW): ") for k in range(1, count + 1)]


# ------------------------------------------------------------------ core maths
def diversity_factor(individual_maxima, group_max):
    """DF = Sum of individual maximum demands / Group maximum demand"""
    return sum(individual_maxima) / group_max


def coincidence_factor(individual_maxima, group_max):
    """Coincidence factor = Group maximum demand / Sum of individual maxima"""
    return group_max / sum(individual_maxima)


def group_max_from_df(individual_maxima, df):
    """Group maximum demand = Sum of individual maxima / DF"""
    return sum(individual_maxima) / df


def remark_on_df(df):
    """Short comment on the diversity factor."""
    if df < 1.05:
        return "Almost no diversity: the consumers peak together."
    if df < 1.5:
        return "Moderate diversity."
    return "High diversity: the consumers peak at different times."


# ------------------------------------------------------------------- display
def from_maximum_demands():
    count = get_int_in_range("Number of consumers (2 to 20): ", 2, 20)
    maxima = get_individual_maxima(count)
    total = sum(maxima)

    while True:
        group_max = get_positive_float("Maximum demand of the whole group (kW): ")
        if group_max <= total:
            break
        print(f"  Group maximum demand cannot exceed the sum of individual maxima ({total:.2f} kW).")

    df = diversity_factor(maxima, group_max)
    print("\n  ----- Diversity Factor -----")
    print(f"  Sum of individual maximum demands = {total:.2f} kW")
    print(f"  Group (coincident) maximum demand = {group_max:.2f} kW")
    print(f"  DIVERSITY FACTOR                  = {df:.3f}")
    print(f"  Coincidence factor                = {coincidence_factor(maxima, group_max):.3f}")
    print(f"  Diversity (capacity not needed)   = {total - group_max:.2f} kW  ({(1 - group_max / total) * 100:.1f} % of the sum)")
    print(f"  Remark                            = {remark_on_df(df)}")

    cost = input("Cost of supply capacity per kW (Rs/kW, press Enter to skip): ").strip()
    if cost:
        try:
            saving = (total - group_max) * float(cost)
            print(f"  Capacity cost saved by diversity  = Rs {saving:,.2f}")
        except ValueError:
            print("  Cost not understood, so it was skipped.")


def from_load_curves():
    count = get_int_in_range("Number of consumers (2 to 6): ", 2, 6)
    readings = get_int_in_range("Number of readings in each load curve (2 to 48): ", 2, 48)
    interval = get_positive_float("Time covered by each reading (hours): ")

    curves = []
    print("  Enter each consumer's load readings in kW, separated by commas.")
    for k in range(1, count + 1):
        curves.append(get_readings(f"  Consumer {k} loads: ", readings))

    group = [sum(c[i] for c in curves) for i in range(readings)]
    maxima = [max(c) for c in curves]
    group_max = max(group)
    df = diversity_factor(maxima, group_max)
    peak_at = group.index(group_max) + 1

    print("\n  ----- Load Table (kW) -----")
    header = f"  {'Reading':>8}" + "".join(f"{'C' + str(k):>9}" for k in range(1, count + 1)) + f"{'Group':>10}"
    print(header)
    for i in range(readings):
        row = f"  {i + 1:>8}" + "".join(f"{c[i]:>9.2f}" for c in curves) + f"{group[i]:>10.2f}"
        print(row + ("   <-- group peak" if i == peak_at - 1 else ""))

    print("\n  ----- Results -----")
    for k, (c, m) in enumerate(zip(curves, maxima), start=1):
        print(f"  Consumer {k}: maximum {m:.2f} kW at reading {c.index(m) + 1}, load factor {sum(c) / readings / m:.3f}")
    print(f"\n  Sum of individual maximum demands = {sum(maxima):.2f} kW")
    print(f"  Group maximum demand              = {group_max:.2f} kW (at reading {peak_at})")
    print(f"  DIVERSITY FACTOR                  = {df:.3f}")
    print(f"  Coincidence factor                = {1 / df:.3f}")
    print(f"  Group load factor                 = {sum(group) / readings / group_max:.3f}")
    print(f"  Remark                            = {remark_on_df(df)}")
    print(f"  Average of the individual load factors (weighted) = {sum(sum(c) / readings for c in curves) / sum(maxima):.3f}")
    print("  Group load factor = diversity factor x that weighted average,")
    print("  so diversity makes the group load curve smoother and the plant better used.")


def size_from_diversity():
    count = get_int_in_range("Number of consumers (2 to 20): ", 2, 20)
    maxima = get_individual_maxima(count)
    df = get_factor_at_least_one("Diversity factor of the group (1 or more): ")
    pf = get_power_factor("Power factor of the group load (0 to 1): ")
    growth = get_nonneg_float("Allowance for future growth (%; 0 for none): ")

    total = sum(maxima)
    group_max = group_max_from_df(maxima, df)
    kva = group_max / pf
    kva_growth = kva * (1 + growth / 100)

    print("\n  ----- Supply Equipment Sizing -----")
    print(f"  Sum of individual maximum demands = {total:.2f} kW")
    print(f"  Group maximum demand              = {group_max:.2f} kW")
    print(f"  Required capacity                 = {kva:.2f} kVA")
    if growth > 0:
        print(f"  With {growth:g} % growth allowance     = {kva_growth:.2f} kVA")
    print(f"  Without diversity you would need  = {total / pf:.2f} kVA")
    print(f"  Capacity saved                    = {total / pf - kva:.2f} kVA")


def overall_diversity():
    levels = get_int_in_range("Number of levels (2 to 5): ", 2, 5)
    total = get_positive_float("Sum of consumers' individual maximum demands (kW): ")

    names = ["consumers on a transformer", "transformers on a feeder",
             "feeders on a substation", "substations on the system", "systems in the grid"]
    factors = []
    for k in range(levels):
        factors.append(get_factor_at_least_one(f"  Diversity factor between {names[k]}: "))

    print("\n  ----- Overall Diversity -----")
    demand = total
    overall = 1.0
    print(f"  {'Level':>6}{'DF':>9}{'Demand after (kW)':>20}")
    print(f"  {'start':>6}{'':>9}{demand:>20.2f}")
    for k, df in enumerate(factors, start=1):
        demand /= df
        overall *= df
        print(f"  {k:>6}{df:>9.3f}{demand:>20.2f}")
    print(f"\n  OVERALL DIVERSITY FACTOR = {overall:.3f}  (product of the factors)")
    print(f"  System maximum demand    = {demand:.2f} kW")


def menu():
    print("\n" + "=" * 58)
    print("           DIVERSITY FACTOR CALCULATOR")
    print("=" * 58)
    print(" 1. From individual and group maximum demands")
    print(" 2. From consumer load curves (readings)")
    print(" 3. Size supply equipment using the diversity factor")
    print(" 4. Overall diversity factor over several levels")
    print(" 0. Exit")
    print("-" * 58)


def main():
    while True:
        menu()
        choice = input("Enter your choice: ").strip()

        if choice == "1":
            from_maximum_demands()
        elif choice == "2":
            from_load_curves()
        elif choice == "3":
            size_from_diversity()
        elif choice == "4":
            overall_diversity()
        elif choice == "0":
            print("\nThank you for using the calculator. Goodbye!")
            break
        else:
            print("  Invalid choice. Please select from the menu.")


if __name__ == "__main__":
    main()
