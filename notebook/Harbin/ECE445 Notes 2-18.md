It's time to test electrodes, but we'll need to look deeper into what op-amps are suited for our application.

Some considerations for [sEMG Measurement](https://people.ece.cornell.edu/land/courses/ece5030/labs/f2009/EMG_measurement_and_recording.pdf).
- High common mode rejection ratio (CMRR)
	- eliminate signals common to both electrodes
	- differential amplifier
- Very high input impedance
	- $10^{10}$ GOhm range for dry electrodes
	- or rather [low input bias current](https://e2e.ti.com/blogs_/archives/b/thesignal/posts/i-need-high-input-impedance) $I_B$
- Short distance to the signal source
	- Power line noise is introduced
	- Limit parasitic capacitance
- Strong DC signal suppression

For initial testing with parts available at the ECE Services Shop, might consider the following:

Amplifiers: 
- CA3140E
- MC33071P
-  MC33178
- ouLF347
- TL072
	- Some drift, higher offset voltage, but similar input bias current with slightly lower CMRR
- TL074BCN
- LF412
- LF353
- LF156

TA:
- look into breadboard/circuit for ADC or solderable PCB
- memory on dev board is limited; consider whether or not to train/run model from on-chip or external computer