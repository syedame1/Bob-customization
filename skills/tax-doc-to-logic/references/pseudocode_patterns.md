# Generated from: Notification No. 14/2024 - Central Tax (Rate)
# Decision table: decision_table_notification_14_2024.csv
# Draft for engineer + tax-counsel review. Do not deploy without sign-off.

function determine_gst(transaction, as_of_date):
    # transaction: {
    #   category: one of [renewable_energy_device, educational_service,
    #                      legal_service, restaurant_food_beverage, gta_transport, ...],
    #   sub_condition: string describing the specific scenario flags,
    #   buyer_type: e.g. "business_entity" | "registered_person" | "consumer",
    #   buyer_turnover_above_threshold: boolean,
    #   room_tariff: number (for restaurant/hotel cases),
    #   gta_forward_charge_opted: boolean
    # }
    # as_of_date: date for which the determination applies

    result = { rate: null, exemption: false, reverse_charge: false,
               itc_eligible: null, rule_id: null, source_clause: null }

    # --- R001/R002: Renewable energy devices (date-bound rate change) ---
    if transaction.category == "renewable_energy_device":
        if as_of_date >= 2024-07-01:
            # R001: Entry 25 - solar water heaters/solar power systems at 12%, ITC eligible
            result.rate = 12
            result.itc_eligible = true
            result.rule_id = "R001"
            result.source_clause = "Notification 14/2024, Entry 25"
            return result
        else:
            # R002: pre-2024-07-01 rate under Notification 1/2017
            result.rate = 5
            result.itc_eligible = true
            result.rule_id = "R002"
            result.source_clause = "Notification 14/2024, Entry 25 (ref Notification 1/2017)"
            return result

    # --- R003/R004: Educational services ---
    if transaction.category == "educational_service":
        if transaction.sub_condition == "entrance_exam_fee_separately_charged":
            # R004: Entry 26 - entrance exam fees taxed at 18%
            result.rate = 18
            result.itc_eligible = true
            result.rule_id = "R004"
            result.source_clause = "Notification 14/2024, Entry 26"
            return result
        else:
            # R003: Entry 26 - core education up to higher secondary is exempt
            result.exemption = true
            result.rate = 0
            result.rule_id = "R003"
            result.source_clause = "Notification 14/2024, Entry 26"
            return result

    # --- R005/R006: Legal services (reverse charge dependent on recipient turnover) ---
    if transaction.category == "legal_service":
        # TODO: confirm with tax counsel (R005) — "threshold limit for registration"
        # is referenced but not numerically defined in this notification; the
        # numeric threshold must be sourced from the relevant CGST registration
        # provisions (currently Rs. 20 lakh / Rs. 10 lakh for special category
        # states, subject to change) and injected as a config value, not hardcoded.
        if transaction.buyer_turnover_above_threshold == true:
            # R005: Entry 27 - reverse charge, recipient pays GST
            result.reverse_charge = true
            result.rate = 18
            result.rule_id = "R005"
            result.source_clause = "Notification 14/2024, Entry 27"
            return result
        else:
            # R006: Entry 27 - below threshold, supplier (advocate) pays normally
            result.rate = 18
            result.itc_eligible = true
            result.reverse_charge = false
            result.rule_id = "R006"
            result.source_clause = "Notification 14/2024, Entry 27"
            return result

    # --- R007/R008: Restaurant food and beverage ---
    if transaction.category == "restaurant_food_beverage":
        if transaction.room_tariff is not null and transaction.room_tariff > 7500:
            # R008: Entry 28 - restaurant within hotel, room tariff > Rs 7,500 -> 18% with ITC
            result.rate = 18
            result.itc_eligible = true
            result.rule_id = "R008"
            result.source_clause = "Notification 14/2024, Entry 28"
            return result
        else:
            # R007: Entry 28 - standard restaurant rate 5%, no ITC
            result.rate = 5
            result.itc_eligible = false
            result.rule_id = "R007"
            result.source_clause = "Notification 14/2024, Entry 28"
            return result

    # --- R009/R010: Goods Transport Agency ---
    if transaction.category == "gta_transport":
        if transaction.gta_forward_charge_opted == true:
            # TODO: confirm with tax counsel (R010) — notification states the
            # forward-charge option, once exercised, applies for the entire
            # financial year. This stateful constraint should be enforced at
            # the GTA-registration/config level, not per-transaction; flag if
            # transaction-level data conflicts with the GTA's annual election.
            # R010: Entry 29 - forward charge opted -> 12% with ITC
            result.rate = 12
            result.itc_eligible = true
            result.reverse_charge = false
            result.rule_id = "R010"
            result.source_clause = "Notification 14/2024, Entry 29"
            return result
        else:
            if transaction.buyer_type == "registered_person":
                # R009: Entry 29 - reverse charge by registered recipient, 5% no ITC
                result.rate = 5
                result.itc_eligible = false
                result.reverse_charge = true
                result.rule_id = "R009"
                result.source_clause = "Notification 14/2024, Entry 29"
                return result
            else:
                # TODO: confirm with tax counsel — notification does not explicitly
                # state treatment when recipient is unregistered and GTA has not
                # opted for forward charge. Likely falls back to GTA paying under
                # forward charge by default, but verify against base CGST rate
                # notification rather than assuming.
                result.rate = null
                result.rule_id = "NEEDS_REVIEW_R009_unregistered_recipient"
                result.source_clause = "Notification 14/2024, Entry 29 (gap)"
                return result

    # --- Default / no match ---
    # TODO: no rule in this notification matches; fall through to base rate
    # notification logic (outside scope of this document).
    result.rule_id = "DEFAULT_NOT_COVERED"
    return result
