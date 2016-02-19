#Overflow Row Distribution Plug-in

This plug-in was altered from the sample Overflow Row Distribution plug-in described in the [Pentaho Wiki](http://wiki.pentaho.com/display/EAI/PDI+Row+Distribution+Plugin+Development "PDI Row Distribution Plugin Development").

##Installation

The simplest installation is to place the plug-in jar file in a folder named plugins in the .kettle directory in KETTLE_HOME.

##Usage

To use this plug-in, once it is installed, right-click on the source step, mouse over Data Movement, and select Overflow
from the options. This should update the outgoing hop to display a balanced scale icon, indicating that it is no longer
doing Round-robin.

##Modifications

###Original Code

The original code is shown below. It is a simplistic approach to filling each output RowSet
queue with tasks until the queue reaches its max capacity. By default this value is 100000.
If the Number of copies to start is changed to a value greater than one on the destination step,
this means the first instance of that step will receive all the tasks until its queue is full.
Once the queue is full, the next tasks will be added to the next instance, and so on. This approach
doesn't load balance the work very will in this scenario.

This code is also dangerous, as it appears that it can get stuck in the while loop if the job is
cancelled while it checking for an open queue and none are available.

Only the distributeRow function is shown below.

```Java
@Override
public void distributeRow(RowMetaInterface rowMeta, Object[] row, StepInterface step) throws KettleStepException {

	RowSet rowSet = step.getOutputRowSets().get(step.getCurrentOutputRowSetNr());

	boolean added = false;
	while (!added) {
	  added=rowSet.putRowWait(rowMeta, row, 1, TimeUnit.NANOSECONDS);
	  if (added) {
	    break;
	  }
	  step.setCurrentOutputRowSetNr(step.getCurrentOutputRowSetNr()+1);
	  if (step.getCurrentOutputRowSetNr()>step.getOutputRowSets().size()-1) {
	    step.setCurrentOutputRowSetNr(0);
	  }
	  rowSet = step.getOutputRowSets().get(step.getCurrentOutputRowSetNr());
	}

}

```

###Updated Code

This is the updated code for this plug-in. It attempts to improve upon the original code by distributing
the tasks across all destination steps, and detecting when the Job has been cancelled. All the changes in
the code have been described in the commented code below.

Only the distributeRow function is shown below.

```Java
@Override
public void distributeRow(RowMetaInterface rowMeta, Object[] row, StepInterface step) throws KettleStepException {

	// Get current outputRowSet
	RowSet rowSet = step.getOutputRowSets().get(step.getCurrentOutputRowSetNr());
	
	// Initialize added Boolean
	boolean added = false;
	
	/*
	 * Differently than the example Overflow distribution, check that the step
	 * has not been stopped. Before it was looping until added was true, which
	 * didn't really do anything because as soon as added is true there is a 
	 * break from the loop.
	 */
	while (!step.isStopped()) {

		/*
		 * Differently than the example Overflow distribution, wait a longer
		 * period of time when trying to put the row. This helps reduce some of
		 * the overhead caused by the while loop constantly checking for available
		 * outputRowSets when the Number of rows in the RowSet is small.
		 * 
		 * The number of rows in the RowSet value can be managed within the properties
		 * of the Transformation: Transformation properties -> Miscellaneous -> Nr of rows in rowset
		 * 
		 * 10 seconds was picked arbitrarily. For use-cases with very short running
		 * tasks, a smaller wait time should be used. A longer wait can be used for
		 * longer running tasks.
		 */
		added = rowSet.putRowWait(rowMeta, row, 10, TimeUnit.SECONDS);
		
		/*
		 * Differently than the example Overflow distribution, always increment to the
		 * next output RowSet. This provides more of a round-robin approach, but will
		 * put a row in the next available output RowSet.
		 */
		step.setCurrentOutputRowSetNr(step.getCurrentOutputRowSetNr() + 1);
		if (step.getCurrentOutputRowSetNr() > step.getOutputRowSets().size() - 1) {
			step.setCurrentOutputRowSetNr(0);
		}

		/*
		 * If the row was added to a RowSet, break from the while loop, we'ere done.
		 */
		if (added) {
			break;
		}
		
		/*
		 * If the row was not added to the RowSet, get the next RowSet and try again.
		 */
		rowSet = step.getOutputRowSets().get(step.getCurrentOutputRowSetNr());
		
	}
	
}

```

##Developer Instructions

To modify the source code, clone this project.