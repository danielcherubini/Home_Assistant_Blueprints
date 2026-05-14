# Scheduled Humidity Override Plan

**Goal:** Add a configurable daily time window that turns humidity control ON/OFF, overriding all humidity-based logic but yielding to the manual override switch.

**Architecture:** Three new input fields (enable switch, start time, end time), three new triggers (two time-based + one state-based for mid-window toggling), and three new choose branches inserted after manual override logic. All existing humidity-based branches are gated with a `within_schedule_window` check.

**Tech Stack:** Home Assistant blueprint YAML, Jinja2 templates

---

### Task 1: Add new input fields and update blueprint metadata

**Depends on:** (none — foundational task)

**Context:**
The blueprint needs three new configurable inputs for the scheduled override feature. These follow the existing pattern of optional inputs (empty string = disabled). The blueprint version needs to bump from v1.8 to v1.9 and the description needs to mention the new feature.

**Files:**
- Modify: `smart_humidity_control.yaml`

**What to implement:**

1. Update the blueprint metadata block:
   - Change `name` from `"Smart Humidity Control (v1.8)"` to `"Smart Humidity Control (v1.9)"`
   - Update `description` to mention the scheduled time window feature, e.g.: `"Automatically control humidity in your home based on house mean humidity. Includes special handling for bathroom and 4th floor humidity spikes with configurable offsets, and a scheduled daily time window override."`

2. Add three new input fields. Insert them after the `dashboard_url` input and before the `# Thresholds` comment block:

   ```yaml
   schedule_enable_switch:
       name: Schedule Enable Switch (Optional)
       description: Switch or input boolean that enables/disables the scheduled time window. When ON, the schedule is active. When OFF or not configured, the schedule is ignored.
       default: ""
       selector:
           entity:
               filter:
                   - domain:
                         - switch
                         - input_boolean

   schedule_start_time:
       name: Schedule Start Time
       description: Time of day to turn humidity control ON (schedule must be enabled). Supports overnight windows (e.g., 22:00 start, 06:00 end).
       default: "06:00:00"
       selector:
           time:

   schedule_end_time:
       name: Schedule End Time
       description: Time of day to turn humidity control OFF (schedule must be enabled). Supports overnight windows (e.g., 22:00 start, 06:00 end).
       default: "10:00:00"
       selector:
           time:
   ```

**Steps:**
- [ ] Update `name` field to v1.9
- [ ] Update `description` field to mention scheduled override
- [ ] Add `schedule_enable_switch` input after `dashboard_url`
- [ ] Add `schedule_start_time` input
- [ ] Add `schedule_end_time` input
- [ ] Verify YAML syntax (no indentation errors) — run: `python3 -c "import yaml; yaml.add_constructor('!input', lambda l, n: None); yaml.safe_load(open('smart_humidity_control.yaml')); print('YAML valid')"`
- [ ] Commit with message: "feat: add scheduled time window inputs (v1.9)"

**Acceptance criteria:**
- [ ] Blueprint metadata updated to v1.9 with description mentioning scheduled override
- [ ] Three new input fields added with correct selectors and defaults
- [ ] YAML is valid (no syntax errors)

---

### Task 2: Add new variables and triggers

**Depends on:** Task 1 (references `!input schedule_*` fields)

**Context:**
The new inputs need corresponding variables for use in conditions/templates. The `within_schedule_window` template must handle both same-day windows (06:00→10:00) and overnight windows (22:00→06:00) using normalized time comparison (`[:5]` to strip seconds). Three new triggers are needed: two time triggers for the window boundaries and one state trigger for mid-window enable/disable toggling.

**Files:**
- Modify: `smart_humidity_control.yaml`

**What to implement:**

1. Add four new variables to the `variables:` block. Insert after the `dashboard_url` variable and before the `house_humidity` variable:

   ```yaml
   schedule_enable_switch: !input schedule_enable_switch
   schedule_start_time: !input schedule_start_time
   schedule_end_time: !input schedule_end_time
   schedule_enabled: "{{ schedule_enable_switch != '' and is_state(schedule_enable_switch, 'on') }}"
   within_schedule_window: >-
     {{
       schedule_enabled and
       (
         (schedule_start_time[:5] <= schedule_end_time[:5] and now().strftime('%H:%M') >= schedule_start_time[:5] and now().strftime('%H:%M') < schedule_end_time[:5]) or
         (schedule_start_time[:5] > schedule_end_time[:5] and (now().strftime('%H:%M') >= schedule_start_time[:5] or now().strftime('%H:%M') < schedule_end_time[:5]))
       )
     }}
   ```

   **Logic explanation for the executing agent:**
   - `schedule_start_time[:5]` strips seconds from `"06:00:00"` → `"06:00"` to match `now().strftime('%H:%M')` format
   - Same-day window (start ≤ end): active when current time is between start and end
   - Overnight window (start > end): active when current time is after start OR before end (e.g., 22:00→06:00 is active from 22:00 through 05:59)

2. Add three new triggers to the `trigger:` block. Insert after the existing triggers:

   ```yaml
   - platform: time
     at: !input schedule_start_time
     id: schedule_on
   - platform: time
     at: !input schedule_end_time
     id: schedule_off
   - platform: state
     entity_id: !input schedule_enable_switch
     id: schedule_enable_change
   ```

**Steps:**
- [ ] Add four new variables to the variables block (schedule_enable_switch, schedule_start_time, schedule_end_time, schedule_enabled, within_schedule_window)
- [ ] Add three new triggers (schedule_on, schedule_off, schedule_enable_change)
- [ ] Verify YAML syntax — run: `python3 -c "import yaml; yaml.add_constructor('!input', lambda l, n: None); yaml.safe_load(open('smart_humidity_control.yaml')); print('YAML valid')"`
- [ ] Commit with message: "feat: add schedule variables and triggers"

**Acceptance criteria:**
- [ ] All four new variables added correctly
- [ ] `within_schedule_window` template handles both same-day and overnight windows
- [ ] Time format normalization uses `[:5]` on both input and now() sides
- [ ] Three new triggers added with correct IDs
- [ ] YAML is valid

---

### Task 3: Add schedule choose branches and gate existing logic

**Depends on:** Task 2 (references `within_schedule_window`, `schedule_enabled` variables and `schedule_*` trigger IDs)

**Context:**
Three new choose branches implement the schedule ON, schedule OFF, and mid-window enable/disable logic. They are inserted AFTER the manual override branches (which have highest priority) and BEFORE all humidity-based branches. All existing humidity-based branches (bathroom, 4th floor, outdoor balance, general) must be gated with a `not within_schedule_window` condition to prevent them from interfering during the schedule window.

**Files:**
- Modify: `smart_humidity_control.yaml`

**What to implement:**

1. **Add Schedule ON branch** — insert as the first branch AFTER the two manual override branches and BEFORE the bathroom priority branch:

   ```yaml
   # Scheduled time window - turn ON
   - conditions:
         - condition: trigger
           id: schedule_on
         - condition: template
           value_template: "{{ schedule_enabled }}"
         - condition: template
           value_template: "{{ override_switch == '' or is_state(override_switch, 'off') }}"
     sequence:
         - service: switch.turn_on
           target:
               entity_id: !input humidity_control_switch
         - service: input_text.set_value
           target:
               entity_id: !input mode_tracker
           data:
               value: "scheduled"
         - if:
               - condition: template
                 value_template: "{{ notification_device != '' }}"
           then:
               - device_id: !input notification_device
                 domain: mobile_app
                 type: notify
                 title: "Smart Humidity"
                 message: "Humidity Control - On (Scheduled)"
                 data:
                     url: "{{ dashboard_url if dashboard_url != '' else '/lovelace' }}"
                     clickAction: "{{ dashboard_url if dashboard_url != '' else '/lovelace' }}"
   ```

2. **Add Schedule OFF branch** — insert immediately after the Schedule ON branch:

   ```yaml
   # Scheduled time window - turn OFF
   - conditions:
         - condition: trigger
           id: schedule_off
         - condition: template
           value_template: "{{ schedule_enabled }}"
         - condition: template
           value_template: "{{ override_switch == '' or is_state(override_switch, 'off') }}"
     sequence:
         - service: switch.turn_off
           target:
               entity_id: !input humidity_control_switch
         - service: input_text.set_value
           target:
               entity_id: !input mode_tracker
           data:
               value: "idle"
         - if:
               - condition: template
                 value_template: "{{ notification_device != '' }}"
           then:
               - device_id: !input notification_device
                 domain: mobile_app
                 type: notify
                 title: "Smart Humidity"
                 message: "Humidity Control - Off (Scheduled)"
                 data:
                     url: "{{ dashboard_url if dashboard_url != '' else '/lovelace' }}"
                     clickAction: "{{ dashboard_url if dashboard_url != '' else '/lovelace' }}"
   ```

3. **Add Schedule Enable Change branch** — insert immediately after the Schedule OFF branch.

   **Critical:** The `schedule_enabled` condition is NOT at the top level — it's inside the nested choose. This ensures the branch fires when the user disables the schedule mid-window (otherwise `not schedule_enabled` would block the branch and the system would stay stuck ON).

   ```yaml
   # Schedule enable switch changed mid-window
   - conditions:
         - condition: trigger
           id: schedule_enable_change
         - condition: template
           value_template: "{{ override_switch == '' or is_state(override_switch, 'off') }}"
     sequence:
         - choose:
               # Schedule enabled and within window - turn ON
               - conditions:
                     - condition: template
                       value_template: "{{ schedule_enabled and within_schedule_window }}"
                 sequence:
                     - service: switch.turn_on
                       target:
                           entity_id: !input humidity_control_switch
                     - service: input_text.set_value
                       target:
                           entity_id: !input mode_tracker
                       data:
                           value: "scheduled"
                     - if:
                           - condition: template
                             value_template: "{{ notification_device != '' }}"
                         then:
                             - device_id: !input notification_device
                               domain: mobile_app
                               type: notify
                               title: "Smart Humidity"
                               message: "Humidity Control - On (Schedule Enabled)"
                               data:
                                   url: "{{ dashboard_url if dashboard_url != '' else '/lovelace' }}"
                                   clickAction: "{{ dashboard_url if dashboard_url != '' else '/lovelace' }}"
               # Schedule enabled but outside window - turn OFF
               - conditions:
                     - condition: template
                       value_template: "{{ schedule_enabled and not within_schedule_window }}"
                 sequence:
                     - service: switch.turn_off
                       target:
                           entity_id: !input humidity_control_switch
                     - service: input_text.set_value
                       target:
                           entity_id: !input mode_tracker
                       data:
                           value: "idle"
                     - if:
                           - condition: template
                             value_template: "{{ notification_device != '' }}"
                         then:
                             - device_id: !input notification_device
                               domain: mobile_app
                               type: notify
                               title: "Smart Humidity"
                               message: "Humidity Control - Off (Outside Schedule Window)"
                               data:
                                   url: "{{ dashboard_url if dashboard_url != '' else '/lovelace' }}"
                                   clickAction: "{{ dashboard_url if dashboard_url != '' else '/lovelace' }}"
               # Schedule disabled - turn OFF (handles mid-window disable)
               - conditions:
                     - condition: template
                       value_template: "{{ not schedule_enabled }}"
                 sequence:
                     - service: switch.turn_off
                       target:
                           entity_id: !input humidity_control_switch
                     - service: input_text.set_value
                       target:
                           entity_id: !input mode_tracker
                       data:
                           value: "idle"
                     - if:
                           - condition: template
                             value_template: "{{ notification_device != '' }}"
                         then:
                             - device_id: !input notification_device
                               domain: mobile_app
                               type: notify
                               title: "Smart Humidity"
                               message: "Humidity Control - Off (Schedule Disabled)"
                               data:
                                   url: "{{ dashboard_url if dashboard_url != '' else '/lovelace' }}"
                                   clickAction: "{{ dashboard_url if dashboard_url != '' else '/lovelace' }}"
   ```

4. **Gate all existing humidity-based branches** — add the following condition as the FIRST condition in each of the existing humidity-based choose branches (bathroom on/off, 4th floor on/off, outdoor balance on/off, general). Insert it AFTER the existing trigger id and override switch conditions:

   ```yaml
   - condition: template
     value_template: "{{ not within_schedule_window }}"
   ```

   **Specific branches to modify (7 total):**
   - Bathroom priority - turn on (add after `override_switch` condition)
   - Bathroom priority - turn off (add after `override_switch` condition)
   - 4th Floor priority - turn on (add after `override_switch` condition)
   - 4th Floor priority - turn off (add after `override_switch` condition)
   - Outdoor Balance - turn on (add after `override_switch` condition)
   - Outdoor Balance - turn off (add after `override_switch` condition)
   - **General humidity control — outer conditions** (add after `override_switch` condition)

   **Do NOT add this gate to:** the manual override branches (they have highest priority).

5. **Fix general turn-off mode list** — in the general humidity control turn-off branch, update the mode list to include `"scheduled"`:

   Change:
   ```yaml
   value_template: "{{ current_mode in ['general', 'idle', 'outdoor_balance', 'bathroom', 'fourth_floor'] }}"
   ```
   To:
   ```yaml
   value_template: "{{ current_mode in ['general', 'idle', 'outdoor_balance', 'bathroom', 'fourth_floor', 'scheduled'] }}"
   ```

   **Note on turn-on mode list:** The general turn-on mode list (`['idle', 'outdoor_balance']`) is intentionally NOT updated to include `"scheduled"`. The schedule handles its own activation via the schedule branches, and the turn-on branch can't fire when mode is `"scheduled"` because the switch is already on (the `state: "off"` condition blocks it).

**Steps:**
- [ ] Add Schedule ON choose branch after manual override branches
- [ ] Add Schedule OFF choose branch after Schedule ON
- [ ] Add Schedule Enable Change choose branch with nested choose for within/outside window
- [ ] Add `not within_schedule_window` gate to all 7 humidity-based choose branches (including General outer conditions)
- [ ] Add `"scheduled"` to general turn-off mode list
- [ ] Verify YAML syntax (indentation is critical in nested choose blocks) — run: `python3 -c "import yaml; yaml.add_constructor('!input', lambda l, n: None); yaml.safe_load(open('smart_humidity_control.yaml')); print('YAML valid')"`
- [ ] Commit with message: "feat: implement schedule choose branches and gate humidity logic"

**Acceptance criteria:**
- [ ] Schedule ON branch turns switch on, sets mode to "scheduled", sends notification
- [ ] Schedule OFF branch turns switch off, sets mode to "idle", sends notification
- [ ] Schedule Enable Change branch handles mid-window enable (turns on if within window, off if outside)
- [ ] All schedule branches check override guard (manual override takes priority)
- [ ] All 7 humidity-based branches gated with `not within_schedule_window` (including General)
- [ ] General turn-off mode list includes "scheduled"
- [ ] YAML is valid with correct indentation throughout

---

### Task 4: Final verification and cleanup

**Depends on:** Tasks 1–3

**Context:**
After all changes are implemented, verify the complete blueprint is valid YAML and the logic flows correctly. Check that the priority chain is correct: Manual Override > Schedule > Humidity Logic.

**Files:**
- Modify: `smart_humidity_control.yaml` (if any fixes needed)

**What to implement:**
No new code — verification only.

**Steps:**
- [ ] Run `python3 -c "import yaml; yaml.add_constructor('!input', lambda l, n: None); yaml.safe_load(open('smart_humidity_control.yaml')); print('YAML valid')"` to validate YAML syntax
- [ ] Verify the choose branch order is: (1) Manual Override ON, (2) Manual Override OFF, (3) Schedule ON, (4) Schedule OFF, (5) Schedule Enable Change, (6) Bathroom ON, (7) Bathroom OFF, (8) 4th Floor ON, (9) 4th Floor OFF, (10) Outdoor Balance ON, (11) Outdoor Balance OFF, (12) General
- [ ] Verify all three schedule branches have the override guard condition
- [ ] Verify all 7 humidity-based branches have the `not within_schedule_window` gate (including General)
- [ ] Verify `within_schedule_window` template uses `[:5]` normalization on both sides
- [ ] Verify Schedule Enable Change branch handles three cases: enabled+in-window, enabled+out-of-window, disabled
- [ ] Commit with message: "chore: verify scheduled override implementation"

**Acceptance criteria:**
- [ ] YAML validates without errors
- [ ] Choose branch order matches priority chain
- [ ] All guards and gates are in place
- [ ] Schedule Enable Change handles mid-window disable correctly
- [ ] No regressions to existing functionality
