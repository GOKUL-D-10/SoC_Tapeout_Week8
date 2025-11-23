# Week 8: Layout-Based STA and Timing Evaluation over Various PVT Corners — VSDBabySoC

After routing, it's crucial to verify the timing characteristics of VSDBabySoC across all significant Process, Voltage, and Temperature (PVT) conditions using post-route Static Timing Analysis (STA). Routing introduces actual wire resistance and capacitance, so analyzing the design after these physical effects are present is vital to ensure timing objectives are robustly met. Parasitic-aware timing checks reveal real delays, clock discrepancies, and possible violations that may go unnoticed during preliminary, pre-route STA phases.

The STA tool leverages several Liberty format timing library models, each representing a particular PVT scenario. By automating the timing assessment for all corners, one can extract metrics such as Worst Negative Slack (WNS), Total Negative Slack (TNS), and the most critical setup/hold slack values.

## Preparation of Analysis Files

To carry out post-route STA, the **`sta_across_pvt_route.tcl`** script is required. This script originates from OpenSTA’s example **`sta_across_pvt.tcl`** (in **`OpenSTA/examples/BabySoC`**) and is customized to handle the routed netlist along with the corresponding parasitic data from the SPEF file for a detailed timing evaluation.

Example **`sta_across_pvt_route.tcl`** content:

`tclset list_of_lib_files(1) "sky130_fd_sc_hd__tt_025C_1v80.lib"
set list_of_lib_files(2) "sky130_fd_sc_hd__ff_100C_1v65.lib"
...
set list_of_lib_files(13) "sky130_fd_sc_hd__ss_n40C_1v76.lib"

read_liberty /home/gokul/OpenSTA/examples/timing_libs/avsdpll.lib
read_liberty /home/gokul/OpenSTA/examples/timing_libs/avsddac.lib

for {set i 1} {$i <= [array size list_of_lib_files]} {incr i} {
    *# Load corner-specific Liberty*
    read_liberty /home/gokul/OpenSTA/examples/timing_libs/skywater-pdk-libs-sky130_fd_sc_hd/$list_of_lib_files($i)
    *# Import routed gate-level netlist*
    read_verilog /home/gokul/OpenROAD-flow-scripts/flow/designs/sky130hd/vsdbabysoc/vsdbabysoc_post_place.v
    *# Link the top module*
    link_design vsdbabysoc
    current_design
    *# Apply constraints*
    read_sdc /home/gokul/OpenSTA/examples/BabySoC/vsdbabysoc_post_cts.sdc
    *# Attach extracted parasitics*
    read_spef /home/gokul/OpenROAD-flow-scripts/flow/designs/sky130hd/vsdbabysoc/vsdbabysoc.spef

    *# Execute timing checks*
    check_setup -verbose

    *# Save detailed setup and path analysis*
    report_checks -path_delay min_max -fields {nets cap slew input_pins fanout} -digits {4} > /home/gokul/OpenSTA/examples/BabySoC/STA_OUTPUT/route/min_max_$list_of_lib_files($i).txt

    *# Collect worst slack values and summary metrics*
    exec echo "$list_of_lib_files($i)" >> /home/gokul/OpenSTA/examples/BabySoC/STA_OUTPUT/route/sta_worst_max_slack.txt
    report_worst_slack -max -digits {4} >> /home/gokul/OpenSTA/examples/BabySoC/STA_OUTPUT/route/sta_worst_max_slack.txt

    exec echo "$list_of_lib_files($i)" >> /home/gokul/OpenSTA/examples/BabySoC/STA_OUTPUT/route/sta_worst_min_slack.txt
    report_worst_slack -min -digits {4} >> /home/gokul/OpenSTA/examples/BabySoC/STA_OUTPUT/route/sta_worst_min_slack.txt

    exec echo "$list_of_lib_files($i)" >> /home/gokul/OpenSTA/examples/BabySoC/STA_OUTPUT/route/sta_tns.txt
    report_tns -digits {4} >> /home/gokul/OpenSTA/examples/BabySoC/STA_OUTPUT/route/sta_tns.txt

    exec echo "$list_of_lib_files($i)" >> /home/gokul/OpenSTA/examples/BabySoC/STA_OUTPUT/route/sta_wns.txt
    report_wns -digits {4} >> /home/gokul/OpenSTA/examples/BabySoC/STA_OUTPUT/route/sta_wns.txt
}`

## SDC Constraints for STA

Timing constraints are defined in **`vsdbabysoc_post_cts.sdc`**, which must be updated from the output of the clock tree synthesis process (**`4_cts.sdc`**) and positioned under **`OpenSTA/examples/BabySoC/`**. Example SDC content:

`tclcurrent_design vsdbabysoc
create_clock -name clk -period 11.0000 [get_pins {pll/CLK}]
set_propagated_clock [get_clocks {clk}]
*# Additional constraint lines can be appended here as required*`

## STA Execution Process

The static timing analysis for the routed layout can then be started by running:

`bashsta /home/gokul/OpenSTA/examples/BabySoC/sta_across_pvt_route.tcl`

For Dockerized workflows:

`bashdocker run -it -v $HOME:/data opensta /OpenSTA/examples/BabySoC/sta_across_pvt_route.tcl`

When complete, outputs are generated within:

`text/OpenSTA/examples/BabySoC/STA_OUTPUT/route/`

Key files produced include:

- Detailed min/max path reports
- WNS (Worst Negative Slack)
- TNS (Total Negative Slack)
- Slack summaries for setup/hold checks

These documents provide comprehensive insights verifying whether the design adheres to the timing specifications across all modeled PVT cases. If violations remain, further corrective actions such as logic resizing or routing optimization may be necessary.

---

## **Comparison: Post-Synthesis (Week 3) vs Post-Route (Week 8)**

Timing results for both phases can be visualized—refer to the corresponding table and graphical charts for slacks and violations.

## **Setup Slack**

After incorporating actual routing, **setup slack improves notably in all corners**.

- Fast process corners display increases of **2.5–4.3 ns**
- The typical corner jumps by about **4 ns**
- Slow process corners improve most, rising **6–27 ns**

Generally, post-route analysis relieves many setup violations, particularly in worst-case slow corners.

## **Hold Slack**

Most corners register **slight positive gains** in hold slack, ranging from **0.01–0.02 ns**.

- Only two corners observe minor reductions, but changes (under 0.025 ns) are negligible for practical closure.

Overall, hold paths remain essentially stable post-routing.

## **WNS (Worst Negative Slack)**

Following routing, corners previously failing now meet timing (WNS = 0), while slow corners exhibit significant advances (e.g., **from −55 ns to −28 ns**).

## **TNS (Total Negative Slack)**

TNS demonstrates **dramatic improvement**:

- Massive reductions in total negative slack across all previously failing corners.
- Certain worst cases transition from thousands of violative paths (**−47920, −29705, −8815**) to far smaller numbers or zero.

This reflects a substantial sweep of timing violations after parasitic extraction and rerouting.

## **Summary of Trend**

- **Setup**: Marked improvements everywhere
- **Hold**: Small positive changes, almost no negative impact
- **WNS/TNS**: Major recovery, especially for slow corners and formerly critical paths

---

## **Explanation of Timing Changes (Pre-route vs Post-route)**

## **Root Cause for Timing Shifts**

Post-synthesis figures rely on generalized wire-load models (estimates), while after routing, concrete wiring and extracted parasitics dictate timing. This difference in path estimation arises from:

- Measured wire lengths
- Specific metal layer assignments and via counts
- True net topology and congestion

Thus, post-routing analysis yields results that frequently diverge—often improving for well-routed nets, sometimes worsening for heavily burdened/intricate paths.

## **Role of SPEF Annotation**

The SPEF file (Standard Parasitic Exchange Format) contains detailed RC extraction for each net. When STA tools annotate netlists with SPEF:

- The delay model includes **actual resistances and capacitances** encountered in the layout
- Timing calculations reflect **real arrival times, slew rates, and critical path delays**

Accordingly, paths with efficient routing reveal better slacks, while wires with extra length/capacitance/routing detours may experience delay increases.

## **Impact of Physical Factors**

1. **Capacitance**
    - Longer routes raise ground capacitance.
    - Closely packed wires increase coupling capacitance.
    - Excess capacitance results in slower transitions, greater delays, and weaker setup margins.
2. **Resistance**
    - Thin metals and multi-via wires add resistance.
    - Higher resistance amplifies RC time constants, degrading setup slack.
3. **Coupling**
    - Adjacent traces generate crosstalk, causing additional delays or glitches, which can affect both setup and hold performance depending on switching activity.

Collectively, these parasitic effects define the observed differences in post-route timing and the necessity of thorough analysis.

---

## **Key Point Summary**

- Post-routing checks utilize **precise RC parasitic data**, unlike initial estimates.
- SPEF annotation reshapes path delays using the true physical net topology.
- Factors such as capacitance, resistance, and coupling become the primary influencers of final timing closure, rendering post-route STA a critical step in the design signoff flow.
