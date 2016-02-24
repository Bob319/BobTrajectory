# BobTrajectory

A library that allows for on the fly trajectory generation. 

The build is a single jar that get's deployed to the RoboRio and the Trajectory Server.

## Server Setup 

On the Trajectory Server run via the command line by calling `java -jar bob-trajectory.jar`

## RoboRio Setup

Import the jar into your eclipse project into a lib directory.
Right Click the Jar > Build Path > Add to Build Path

Also include the following in your build.properties file:
lib.dir=${src.dir}/../lib

userLibs=${lib.dir}/bob-library-0.0.1.jar;${lib.dir}/bob-trajectory.jar

## FIRST Robot Code
In robot init start the clients:

    try{
        WaypointClient.start("10.3.19.21");
        TrajectoryClient.start("10.3.19.21");
    }catch(Exception e){
  	  e.printStackTrace();
    }


Create a command group and add two sequential commands
-Build Spline
-Drive Spline


### Command Examples

#### Command Group

    public class DriveAutoSpline extends CommandGroup{
    	public DriveAutoSpline() {
    		//pass a set a of waypoints to the sever
    		//wait for response
    		//load trajectory into drivetrain
    		addSequential(new BuildSingleTowerSpline());
    		
    		//run the trajectory (stored in the drivetrain from the server) 
    		addSequential(new DriveSpline());
    	}
    }

#### Build Spline

    public class BuildDriveStraightSpline extends Command implements ITrajectoryChangeListener{
    	
    	private static final double BACK_OFF = 3.5;
    	private boolean waitingForTrajectory = true;
    	
    	public BuildDriveStraightSpline() {
    		requires(Robot.driveTrain);
    	}
    
    	@Override
    	protected void initialize() {
    		// TODO Auto-generated method stub
    		TrajectoryManager.getInstance().registerListener(this);
    		
    		List<Waypoint> waypoints = new ArrayList<Waypoint>();
    		waypoints.add(new Waypoint(0,0,0));		
    		waypoints.add(new Waypoint(10,0,0));
    		waypoints.add(new Waypoint(16,-2,0));
    		
    		WaypointList waypointList = new WaypointList(waypoints);
    		WaypointManager.getInstance().setWaypointList(waypointList, null);
    	    
    	}
    	@Override
    	protected void execute() {
    		// TODO Auto-generated method stub
    		
    	}
    
    	@Override
    	protected boolean isFinished() {
    		return !waitingForTrajectory;
    	}
    
    	@Override
    	protected void end() {
    		// TODO Auto-generated method stub
    		TrajectoryManager.getInstance().unregisterListener(this);
    		waitingForTrajectory = true;
    	}
    
    	@Override
    	protected void interrupted() {
    		// TODO Auto-generated method stub
    		
    	}
    	
    	
    	@Override
    	public void onTrajectoryChange(CombinedSrxMotionProfile combined, TrajectoryServletSocket source) {
    		System.out.println("Got Trajectory");
    		Robot.driveTrain.setCurrentProfile(combined);
    		waitingForTrajectory = false;
    	}
    
    }

#### Drive Spline

    public class DriveSpline extends Command{
	
        	private long startTime = 0;
        	private long maxTime = 0;
        	
        	int loops = 0;
        
        	//CANTalon rightDriveLead = RobotMap.driveTrainrightDriveLead;
        	//CANTalon leftDriveLead = RobotMap.driveTrainleftDriveLead;
        	
        	boolean motionProfileStarted = true;
        	//true indicates left profile
        	GeneratedMotionProfile leftProfile = new GeneratedMotionProfile(RobotMap.driveTrainleftDriveLead, true);
        	//false indicates right profile
        	GeneratedMotionProfile rightProfile = new GeneratedMotionProfile(RobotMap.driveTrainrightDriveLead, false);
        	
        	
        	public DriveSpline() {
        		requires(Robot.driveTrain);
        	}
        
        	@Override
        	protected void initialize() {
        		/**
        		//calculate the time it should take , used as an override to end the MP command based on a time
        		 
        		int leftCount = Robot.driveTrain.getCurrentProfile().getLeftProfile().getNumPoints();
        		int rightCount = Robot.driveTrain.getCurrentProfile().getRightProfile().getNumPoints();
        		
        		maxTime = Math.max(leftCount, rightCount) * 10; //in ms
        		startTime = System.currentTimeMillis();
        		**/
        		
        		Robot.driveTrain.shiftDown();
        	    loops = 0;
        	    
        	    rightProfile.reset();
        	    leftProfile.reset();
        	    
        	    motionProfileStarted = true;
        	}
        
        	@Override
        	protected void execute() {
        		rightProfile.control();
            	leftProfile.control();
        		
            	RobotMap.driveTrainrightDriveLead.changeControlMode(TalonControlMode.MotionProfile);
            	RobotMap.driveTrainleftDriveLead.changeControlMode(TalonControlMode.MotionProfile);
            	
            	CANTalon.SetValueMotionProfile setRightOutput = rightProfile.getSetValue();
                CANTalon.SetValueMotionProfile setLeftOutput = leftProfile.getSetValue();
            	
                RobotMap.driveTrainrightDriveLead.set(setRightOutput.value);
            	RobotMap.driveTrainleftDriveLead.set(setLeftOutput.value);
            	
            	if(motionProfileStarted){
                	rightProfile.startMotionProfile();
                	leftProfile.startMotionProfile();
                		
                	motionProfileStarted = false;
            	}
        	}
        
        	@Override
        	protected boolean isFinished() {
        		//for testing purposes
        		/**
        		if(true){
        			return true;
        		}
        		**/
        		/**
        		if(System.currentTimeMillis() - startTime > maxTime){
        			return true;
        		}else 
        		**/
        			
        		if( rightProfile.getTimeoutCnt() >2 || leftProfile.getTimeoutCnt() >2){
        	    	return true;
            	}else if (rightProfile.isFinished()==true && leftProfile.isFinished()==true){
        	        return true;
            	}else{
        	    	return false;
            	}
        	}
        
        	@Override
        	protected void end() {
        		RobotMap.driveTrainrightDriveLead.changeControlMode(TalonControlMode.PercentVbus);
            	RobotMap.driveTrainrightDriveFollow.changeControlMode(TalonControlMode.Follower);
            	RobotMap.driveTrainrightDriveFollow.set(RobotMap.driveTrainrightDriveLead.getDeviceID());
             	RobotMap.driveTrainleftDriveLead.changeControlMode(TalonControlMode.PercentVbus);
             	RobotMap.driveTrainleftDriveFollow.changeControlMode(TalonControlMode.Follower);
             	RobotMap.driveTrainleftDriveFollow.set(RobotMap.driveTrainleftDriveLead.getDeviceID());
            	
            	
            	RobotMap.driveTrainleftDriveLead.set(0);
            	RobotMap.driveTrainrightDriveLead.set(0);
            	
            	rightProfile.reset();
            	leftProfile.reset();
            	
            	//Our profile has been run, so let's empty it
            	Robot.driveTrain.setCurrentProfile(null);
        	}
        
        	@Override
        	protected void interrupted() {
        		// TODO Auto-generated method stub
        	}
        
        }
        
        
#### Generated Motion Profile
	public class GeneratedMotionProfile {
	
		/**
		 * The status of the motion profile executer and buffer inside the Talon.
		 * Instead of creating a new one every time we call getMotionProfileStatus,
		 * keep one copy.
		 */
		private CANTalon.MotionProfileStatus _status = new CANTalon.MotionProfileStatus();
	
		/**
		 * reference to the talon we plan on manipulating. We will not changeMode()
		 * or call set(), just get motion profile status and make decisions based on
		 * motion profile.
		 */
		private CANTalon _talon;
		/**
		 * State machine to make sure we let enough of the motion profile stream to
		 * talon before we fire it.
		 */
		private int _state = 0;
		/**
		 * Any time you have a state machine that waits for external events, its a
		 * good idea to add a timeout. Set to -1 to disable. Set to nonzero to count
		 * down to '0' which will print an error message. Counting loops is not a
		 * very accurate method of tracking timeout, but this is just conservative
		 * timeout. Getting time-stamps would certainly work too, this is just
		 * simple (no need to worry about timer overflows).
		 */
		private int _loopTimeout = -1;
		/**
		 * If start() gets called, this flag is set and in the control() we will
		 * service it.
		 */
		private boolean _bStart = false;
	
		/**
		 * Since the CANTalon.set() routine is mode specific, deduce what we want
		 * the set value to be and let the calling module apply it whenever we
		 * decide to switch to MP mode.
		 */
		private CANTalon.SetValueMotionProfile _setValue = CANTalon.SetValueMotionProfile.Disable;
		/**
		 * How many trajectory points do we wait for before firing the motion
		 * profile.
		 */
		private static final int kMinPointsInTalon = 5;
		/**
		 * Just a state timeout to make sure we don't get stuck anywhere. Each loop
		 * is about 20ms.
		 */
		private static final int kNumLoopsTimeout = 10;
		
		/**
		 * Lets create a periodic task to funnel our trajectory points into our talon.
		 * It doesn't need to be very accurate, just needs to keep pace with the motion
		 * profiler executer.  Now if you're trajectory points are slow, there is no need
		 * to do this, just call _talon.processMotionProfileBuffer() in your teleop loop.
		 * Generally speaking you want to call it at least twice as fast as the duration
		 * of your trajectory points.  So if they are firing every 20ms, you should call 
		 * every 10ms.
		 */
		class PeriodicRunnable implements java.lang.Runnable {
		    public void run() {  _talon.processMotionProfileBuffer();    }
		}
		Notifier _notifer = new Notifier(new PeriodicRunnable());
		
		private boolean _bFin = false;
		private int _timeouts = 0;
	
		private boolean isLeft;
		
	
		/**
		 * C'tor
		 * 
		 * @param talon
		 *            reference to Talon object to fetch motion profile status from.
		 * @param isLeft 
		 * 				whether or not is the left talon          
		 */
		public GeneratedMotionProfile(CANTalon talon, boolean isLeft) {
			_talon = talon;
			/*
			 * since our MP is 10ms per point, set the control frame rate and the
			 * notifer to half that
			 */
			_talon.changeMotionControlFramePeriod(5);
			_notifer.startPeriodic(0.005);
			
			this.isLeft  = isLeft;
			
		}
	
		/**
		 * Called to clear Motion profile buffer and reset state info during
		 * disabled and when Talon is not in MP control mode.
		 */
		public void reset() {
			/*
			 * Let's clear the buffer just in case user decided to disable in the
			 * middle of an MP, and now we have the second half of a profile just
			 * sitting in memory.
			 */
			_talon.clearMotionProfileTrajectories();
			/* When we do re-enter motionProfile control mode, stay disabled. */
			_setValue = CANTalon.SetValueMotionProfile.Disable;
			/* When we do start running our state machine start at the beginning. */
			_state = 0;
			_loopTimeout = -1;
			/*
			 * If application wanted to start an MP before, ignore and wait for next
			 * button press
			 */
			_bStart = false;
		}
	
		/**
		 * Called every loop.
		 */
		public void control() {
			/* Get the motion profile status every loop */
			_talon.getMotionProfileStatus(_status);
	
			/*
			 * track time, this is rudimentary but that's okay, we just want to make
			 * sure things never get stuck.
			 */
			if (_loopTimeout < 0) {
				/* do nothing, timeout is disabled */
			} else {
				/* our timeout is nonzero */
				if (_loopTimeout == 0) {
					/*
					 * something is wrong. Talon is not present, unplugged, breaker
					 * tripped
					 */
					instrumentation.OnNoProgress();
					++_timeouts;
				} else {
					--_loopTimeout;
				}
			}
	
			/* first check if we are in MP mode */
			if (_talon.getControlMode() != TalonControlMode.MotionProfile) {
				/*
				 * we are not in MP mode. We are probably driving the robot around
				 * using gamepads or some other mode.
				 */
				_state = 0;
				_loopTimeout = -1;
			} else {
				/*
				 * we are in MP control mode. That means: starting Mps, checking Mp
				 * progress, and possibly interrupting MPs if thats what you want to
				 * do.
				 */
				switch (_state) {
					case 0: /* wait for application to tell us to start an MP */
						if (_bStart) {
							_bStart = false;
		
							_setValue = CANTalon.SetValueMotionProfile.Disable;
							startFilling();
							/*
							 * MP is being sent to CAN bus, wait a small amount of time
							 */
							_state = 1;
							_loopTimeout = kNumLoopsTimeout;
						}
						break;
					case 1: /*
							 * wait for MP to stream to Talon, really just the first few
							 * points
							 */
						/* do we have a minimum numberof points in Talon */
						if (_status.btmBufferCnt > kMinPointsInTalon) {
							/* start (once) the motion profile */
							_setValue = CANTalon.SetValueMotionProfile.Enable;
							/* MP will start once the control frame gets scheduled */
							_state = 2;
							_loopTimeout = kNumLoopsTimeout;
						}
						break;
					case 2: /* check the status of the MP */
						/*
						 * if talon is reporting things are good, keep adding to our
						 * timeout. Really this is so that you can unplug your talon in
						 * the middle of an MP and react to it.
						 */
						if (_status.isUnderrun == false) {
							_loopTimeout = kNumLoopsTimeout;
						}
						/*
						 * If we are executing an MP and the MP finished, start loading
						 * another. We will go into hold state so robot servo's
						 * position.
						 */
						if (_status.activePointValid && _status.activePoint.isLastPoint) {
							/*
							 * because we set the last point's isLast to true, we will
							 * get here when the MP is done
							 */
							_setValue = CANTalon.SetValueMotionProfile.Hold;
							_state = 0;
							_loopTimeout = -1;
							_bFin= true;
							System.out.println("left entered hold");
						}
						break;
				}
			}
			/* printfs and/or logging */
			instrumentation.process(_status);
		}
	
		/** Start filling the MPs to all of the involved Talons. */
		private void startFilling() {
			/* since this example only has one talon, just update that one */
			//startFilling(RightExampleMotionProfile.Points, RightExampleMotionProfile.kNumPoints);
			//startFilling(GeneratedMotionProfileLeft.Points,GeneratedMotionProfileLeft.kNumPoints);
			
			if(isLeft){
				startFilling(Robot.driveTrain.getCurrentProfile().getLeftProfile().getPoints(), 
						Robot.driveTrain.getCurrentProfile().getLeftProfile().getNumPoints());
			}else{
				startFilling(Robot.driveTrain.getCurrentProfile().getRightProfile().getPoints(), 
						Robot.driveTrain.getCurrentProfile().getRightProfile().getNumPoints());
			}
		
		}
	
		private void startFilling(double[][] profile, int totalCnt) {
	
			/* create an empty point */
			CANTalon.TrajectoryPoint point = new CANTalon.TrajectoryPoint();
	
			/* did we get an underrun condition since last time we checked ? */
			if (_status.hasUnderrun) {
				/* better log it so we know about it */
				instrumentation.OnUnderrun();
				/*
				 * clear the error. This flag does not auto clear, this way 
				 * we never miss logging it.
				 */
				_talon.clearMotionProfileHasUnderrun();
			}
			/*
			 * just in case we are interrupting another MP and there is still buffer
			 * points in memory, clear it.
			 */
			_talon.clearMotionProfileTrajectories();
	
			/* This is fast since it's just into our TOP buffer */
			for (int i = 0; i < totalCnt; ++i) {
				/* for each point, fill our structure and pass it to API */
				point.position = profile[i][0];
				point.velocity = profile[i][1];
				point.timeDurMs = (int) profile[i][2];
				point.profileSlotSelect = 0; /* which set of gains would you like to use? */
				point.velocityOnly = false; /* set true to not do any position
											 * servo, just velocity feedforward
											 */
				point.zeroPos = false;
				if (i == 0)
					point.zeroPos = true; /* set this to true on the first point */
	
				point.isLastPoint = false;
				if ((i + 1) == totalCnt)
					point.isLastPoint = true; /* set this to true on the last point  */
	
				_talon.pushMotionProfileTrajectory(point);
			}
		}
	
		/**
		 * Called by application to signal Talon to start the buffered MP (when it's
		 * able to).
		 */
		public void startMotionProfile() {
			_bStart = true;
			_bFin = false;
			_timeouts = 0;
		}
		
		public boolean isFinished(){
			System.out.println("Left_bFin"+_bFin);
			return _bFin;
			
		}
		
		public int getTimeoutCnt(){
			return _timeouts;
		}
	
		/**
		 * 
		 * @return the output value to pass to Talon's set() routine. 0 for disable
		 *         motion-profile output, 1 for enable motion-profile, 2 for hold
		 *         current motion profile trajectory point.
		 */
		public CANTalon.SetValueMotionProfile getSetValue() {
			return _setValue;
		}
	}