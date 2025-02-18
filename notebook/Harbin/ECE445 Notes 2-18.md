It's time to test electrodes, but we'll need to look deeper into what op-amps are suited for our application.

Some considerations for [sEMG Measurement](https://people.ece.cornell.edu/land/courses/ece5030/labs/f2009/EMG_measurement_and_recording.pdf).
- High common mode rejection ratio (CMRR)
	- eliminate signals common to both electrodes
	- differential amplifier
- Very high input impedance
	- $10^{10}$ GOhm range for dry electrodes
- Short distance to the signal source
	- Power line noise is introduced
	- Limit parasitic capacitance
- Strong DC signal suppression

