# Staffflow Project TODO

## Bug Fixes & UX Improvements
- [x] Fix single tick button not working
- [x] Add tooltip for "Tick" metric (time: 15-20 min per tick)
- [x] Add tooltip for "Avg Acuity" (patient condition severity)
- [x] Add tooltip for "Total Patients"
- [x] Add tooltip for "Overloaded Staff"
- [x] Add tooltips for balance status indicators
- [x] Test all tooltips and button functionality
- [x] Create checkpoint

## Graph Convergence & Data Sync Fixes
- [x] Fix workload chart to show clear convergence (lines getting closer together)
- [x] Ensure standard deviation decreases over time (INADEQUATE → SUFFICIENT → IDEAL)
- [x] Fix single tick button to properly advance simulation and update all displays
- [x] Ensure nurse cards update when simulation advances
- [x] Verify simulation state and nurse card data stay synchronized
- [x] Created deliberately imbalanced initialization (all patients to 2-3 nurses)

## UI Refinement
- [x] Integrate new Staffflow logo (staffflow-logo.png)
- [x] Update color theme for better visual consistency
- [x] Ensure all UI elements are accessible and well-laid out
- [x] Update logo in header

## Testing & Delivery
- [x] Test convergence visualization over 20-30 ticks (INADEQUATE→SUFFICIENT→IDEAL)
- [x] Test single tick button functionality (working correctly)
- [x] Verify data synchronization (all queries refetch at 1s interval)
- [x] All 25 tests passing
- [x] Create final checkpoint

## AI Recommendation Improvements
- [x] Fix AI recommendations to provide actual rebalancing solutions during INADEQUATE state
- [x] Ensure recommendations suggest specific patient-to-staff reassignments
- [x] Verify patient-to-staff ratios are manageable after applying suggestions (target 70-90%)
- [x] Test recommendations with LLM integration
- [x] Updated recommendations to use actual simulation state instead of random data
- [x] Improved LLM prompt with detailed workload statistics and specific guidance
- [x] Enhanced fallback recommendations with specific nurse names and utilization percentages
- [x] All 25 tests passing
- [x] Create checkpoint

## Patient-to-Nurse Ratio System
- [x] Replace workload utilization with patient-to-nurse ratio metrics
- [x] Define ratio thresholds: IDEAL (1:1-2), SUFFICIENT (1:3-4), INADEQUATE (1:5+)
- [x] Update balance metric calculation to use patient counts
- [x] Update workload chart to show patient counts instead of utilization %
- [x] Update recommendations to reference patient ratios
- [x] Improved imbalanced initialization (1-2 nurses per unit instead of 2-3)

## Data Stability Fixes
- [x] Remove refetchInterval from all queries (causing constant data regeneration)
- [x] Only refetch data when simulation actually advances (on tick/init/reset)
- [x] Make simulation state persistent and deterministic
- [x] Ensure nurse cards show stable data between ticks
- [x] Test that data only changes when user clicks simulation controls
- [x] Fixed getAssignments to use actual simulation state instead of generating random data
- [x] Fixed getRebalancingSuggestions to use actual simulation state

## Testing & Delivery
- [x] Verify ratio thresholds work correctly (INADEQUATE at 7 patients → IDEAL at 2 patients)
- [x] Test that data stays stable without auto-refetch (confirmed via multiple API calls)
- [x] Ensure simulation still updates on tick/init/reset (convergence working perfectly)
- [x] All 25 tests passing
- [x] Create checkpoint

## Final Polish & Documentation
- [x] Make logo bigger in dashboard header (48px → 80px)
- [x] Create comprehensive README.md
- [x] Export project as zip file (297KB)
- [ ] Deliver to user
