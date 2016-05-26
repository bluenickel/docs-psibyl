#Psibyl Control Integration (Direct)
This page details the process for integrating Psibyl into an execution platform which will run the Psiybl controller.



##Overview

The following sections will decscribe each step required for setting up and running a Psibyl controller from a Visual Studio project.

[TOC]

##Supported Platforms
 The Psibyl controller targets the .Net 4.0 platform and above. If you're project targets .Net 3.5 or below you will need to upgrade it to target .Net 4.0 or higher.

##Dependencies
The Psibyl controller has a dependency on the CentreSpace NMath 5.02 library. These libraries are not included in the Psibyl installer due to their size. You can download the NMath installer from Installation Prerequisites.

##Project assembly references
 There are four Psibyl .Net dlls that must be included in your project. These are,

* Psybil.Common.dll
* Psybil.Core.dll
* Psibyl.Client.dll
* Psibyl.Execution.dll

In addition to these assemblies, you must install the Microsoft Async Nu-Get package. You can do this with the following command in the Package Manager Console.
Install-Package Microsoft.Bcl.Async
This will add the following additional dlls that must be deployed with your application.

* System.Runtime.dll
* System.Threading.Tasks.dll
* System.IO.dll
* Microsoft.Threading.Tasks.dll
* Microsoft.Threading.Tasks.Extensions.Desktop.dll
* Microsoft.Threading.Tasks.Extensions.dll

***Note:*** This only applies when integrating with an application built on .Net 4.0. When using .Net 4.5, you just need to add the assemblies normally in Visual Studio.

##Registering the control platform

In order for clients to be aware of and communicate with controllers, the "control platform" or the application that creates and executes the MPC controller, must register itself with the Psibyl server. Clients are then able to see all available control platforms and the controllers executing on each of these.

The following code snippet demonstrates how to initialize and register the control platform.

`PsibylControl.RegisterControlPlatform("http://localhost:9000/api", "My MPC Control App");`

Note that doing this is optional and may be left out, but if it is left out, the Web UI will not be able to receive any information about the controller.

##Creating a controller
The following code snippets demonstrates how to create a controller. In order to create a controller you must first create or retrieve an MPC Model. This is normally an MPC model created from the Web UI. You will need to include the namespace, `Psibyl.Client` and `Psibyl.Core`.

```
PsibylClient client = new PsibylClient("http://localhost:9000/api");
MPCModel model = client.MPCModels.Get(1);
Controller controller = new Controller(model);
```

You can also retrieve an MPC model usiing the model's name.

`MPCModel model = client.MPCModels.GetByName("My MPC Model");`

Once a controller has been created using the code above, it's state will appear as "Idle" in the Web UI. 

##Managing a controller
There are three main interaction points to be aware of when working with a Psibyl controller. These are Start, Stop and Execute. Starting a controller will initialize the controller from the MPC model putting it into a state where it is ready for execution. The controller will now be in a state "Running". You can start a controller using the following code.

`controller.Start();`

Stopping a controller will unallocate any resources being used by the controller and put it back into the "Idle" state. You can stop a controller using the following code.

`controller.Stop();`

The Execute method must be called on each controller iteration. You must pass the current CV's, MV's, DV's and SP's as double[] parameters. The method will return a double[] as the result. The following code demonstrates usage of the Execute method.

```
double[] cvs = new double[] { 0 };
double[] sps = new double[] { 0 };
double[] mvs = new double[] { 0 };
double[] dvs = new double[] {};

bool run = true;
bool quality = false; //This will contain the quality after the execute method has been called.
int errorId = 0; //This will contain the error id, if any, after the execute method has been called.

while (run)
{
	double[] result = controller.Execute(cvs, sps, dvs, mvs, ref quality, ref errorId); 
	Thread.Sleep(1000);
}
```
Note that the quality and errorId parameters are optional.

##Updating a controller's model
You may change the controller's model at runtime using the following technique. This technique ensures that all clients are aware of the changes made to the model by the controller. The following code shows how to update the controllers model using the UpdateModel method.

```
model.QPMaxIterations = 100;
model.InputCVs[0].Weight = 0.5;
controller.UpdateModel(model);
```

Once UpdateModel has been called the controller will re-initialize, copying the state parameters over from the previous instance. When the controller has finished re-initializing, it swaps itself out with the old instance.

##Updating a CV property
CV's can be updated by specifying the property and an array of values for for each Input CV using the `UpdateInputCVValues` on the controller client. The following code snippet demonstrates how this can be done.

```
Psibyl.Client.PsibylClient client = new Client.PsibylClient("http://localhost:9000/api");
Psibyl.Client.ControllerProxy controller = client.Controllers.GetControllers().First(); 

//For a controller with 2 Input CV's
double[] weightValues = new double[] { 1.5, 2.5 };
client.Controllers.UpdateInputCVValues(controller.Id, "Weight", weightValues);
```

The following is a list of properties that can be used.

* Weight
*	Integrating
*	ConstraintSofteningLowerWeight
*	ConstraintSofteningUpperWeight
*	LowerConstraint
*	LowerConstraintEnabled
*	UpperConstraint
*	UpperConstraintEnabled
*	SoftLowerConstraint
*	SoftLowerConstraintEnabled
*	SoftUpperConstraint
*	SoftUpperConstraintEnabled
*	Enabled
*	UpperRange
*	LowerRange

##Updating a MV property
MV's properties can be updated much the same way as CV's using the UpdateOutputMVValues method on the controller client.

```
Psibyl.Client.PsibylClient client = new Client.PsibylClient("http://localhost:9000/api");
Psibyl.Client.ControllerProxy controller = client.Controllers.GetControllers().First(); 

//For a controller with 2 OutputMVs
double[] values = new double[] { 2, 5 };
client.Controllers.UpdateOutputMVValues(controller.Id, "MaxRateConstraint", values);
```

The following is a list of properties that can be used.

*	MaxRateConstraint
*	MinRateConstraint
*	LowerConstraint
*	LowerConstraintEnabled
*	UpperConstraint
*	UpperConstraintEnabled
*	Weight
*	MinimumStep
*	Enabled
*	UpperRange
*	LowerRange

##Update a Disturbance property
Disturbance properties can be updated much the same way as CV's and MV's using the `UpdateDisturbanceValues` method on the controller client.

```
Psibyl.Client.PsibylClient client = new Client.PsibylClient("http://localhost:9000/api");
Psibyl.Client.ControllerProxy controller = client.Controllers.GetControllers().First();

//For a controller with 1 Disturbance
double[] values = new double[] { 0 };
lient.Controllers.UpdateDisturbanceValues(controller.Id, "Enabled", values);
```

The following is a list of properties that can be used.

* Enabled
* UpperRange
* LowerRange

##Updating MV Gain Multipliers
MV gain multpliers can be updated using the UpdateMVGain method of the controller client. This is acheived by providing a 2D array to the method which represents the model matrix. Each element in the first array represents the Input CV and the elements of these arrays represent the CV/MV pair or step response. The following code snippet demonstrates the use of this method.

```
Psibyl.Client.PsibylClient client = new Client.PsibylClient("http://localhost:9000/api");
Psibyl.Client.ControllerProxy controller = client.Controllers.GetControllers().First(); 

//For a 2 x 2 controller with 2 Input CV's and 2 Output MV's
double[][] gMult = new double[][]
{
	new double[] { 1, 1.5 },
	new double[] { 1.2, 1 }
};

client.Controllers.UpdateMVGain(controller.Id, gMult);
```

##Updating Disturbance Gain Multipliers
Disturbance gains are updated in the same way as MV gains except that the method used is  `UpdateDisturbanceGain` .

##Resetting Predictions
A controllers predictions can be reset using the ResetController method of the controller client. The following snippet demonstrates this.

```
Psibyl.Client.PsibylClient client = new Client.PsibylClient("http://localhost:9000/api");
Psibyl.Client.ControllerProxy controller = client.Controllers.GetControllers().First();
client.Controllers.ResetController(controller.Id );
```

##Accessing prediction results from execution
The following code snippet demostrates how the execution results can be accessed for trending purposes and other analysis.

```
Psibyl.Client.PsibylClient client = new Client.PsibylClient("http://localhost:9000/api");
Psibyl.Client.ControllerProxy controller = client.Controllers.GetControllers().First();

ControllerCurrentValues currentValues = new ControllerCurrentValues()
{
	CVs = new double[] { 0, 0 },
	SPs = new double[] { 0, 0 },
	MVs = new double[] { 0, 0 },
	DVs = new double[] { 0, 0 },
};

ControllerResult result = client.Controllers.ExecuteController(controller.Id, currentValues);

for (int i = 0; i < result.Diagnostic.YPredCLDoubleVector.DoubleVectorValues.Count; i++)
{
	//Vector is packed such that the values alternate between the number of CVs
	int cvIndex = i % result.Diagnostic.Model.InputCVCount;
	string cvName = result.Diagnostic.Model.InputCVsSequenced[cvIndex].Name;
	double value = result.Diagnostic.YPredCLDoubleVector.DoubleVectorValues[i].Value;
	Console.WriteLine($"{cvName}-YPredCL:{value}");
}
for (int i = 0; i < result.Diagnostic.YPredOLDoubleVector.DoubleVectorValues.Count; i++)
{
	//Vector is packed such that the values alternate between the number of CVs
	int cvIndex = i % result.Diagnostic.Model.InputCVCount;
	string cvName = result.Diagnostic.Model.InputCVsSequenced[cvIndex].Name;
	double value = result.Diagnostic.YPredCLDoubleVector.DoubleVectorValues[i].Value;
	Console.WriteLine($"{cvName}-YPredOL:{value}");
}
for (int i = 0; i < result.Diagnostic.PredictedUStepsDoubleVector.DoubleVectorValues.Count; i++)
{
	//Vector is packed such that the values alternate between the number of CVs
	int mvIndex = i % result.Diagnostic.Model.OutputMVCount;
	string mvName = result.Diagnostic.Model.OutputMVsSequenced[mvIndex].Name;
	//Vector is also stepped, i.e each value is duplicated so that it can be trended as a step line.
	double value = result.Diagnostic.PredictedUStepsDoubleVector.DoubleVectorValues[i + 1].Value;
	Console.WriteLine($"{mvName}-PredictedUSteps:{value}");
}
```

##Tracing/Logging
 The Psibyl API provides a way to hook into the trace information which is generated by the controller. This mechanism is based on the System.Diagnostics API provided with the .Net Framework as well as a project called Essential diagnostics. You can obtain more information on these at the following links.
https://msdn.microsoft.com/en-us/library/gg145030(v=vs.110).aspx
https://essentialdiagnostics.codeplex.com/

The Psibyl API provides a simple way for adding Trace Listeners which will be used to log information and errors from the controller. Currently there are three different types of trace listeners that you can add. These are,

*	Event Log Trace Listener - Writes generated messages to the Windows Event log.
*	Rolling File Trace Listener - Writes generated messages to a rolling text file on the file system.
*	Colored Console Trace Listener - Writes generated message to the console window of the application.

The following code demonstrates how to add an Event Log Trace Listener. You will need to add the namespace `Psibyl.Core`.

```
Tracing.AddEventLogTraceListener(Tracing.ExecutionTrace, "My Application - Psibyl MPC");
```

The second parameter is the event log name. This is optional and the default "Psibyl Execution" will be used if not specified. You may use an existing event log or if the event log does not exist, one will be created using the event log name.

##Sample Program
Below is the code for the full sample program used in the previous section.

```
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading;
using Psibyl.Core;
using Psibyl.Core.Model;
using Psibyl.Execution;
namespace Psibyl.Test
{
    class Program
    {
        static void Main(string[] args)
        { //Register the control plarform with the Psibyl Server. 
            PsibylControl.RegisterControlPlatform("http://localhost:9000/api", "My MPC Control App");

            //Add the Event and Console listeners for tracing. 
            Tracing.AddEventLogTraceListener(Tracing.ExecutionTrace, "My Application - Psibyl MPC");
            Tracing.AddColoredConsoleTraceListener(Tracing.ExecutionTrace);

            //Create the Psibyl client which will be used to retrieve the model. 
            Psibyl.Client.PsibylClient client = new Client.PsibylClient("http://localhost:9000/api");

            //Fetch the MPC model using the model name. 
            MPCModel model = client.MPCModels.GetByName("My MPC Model");

            //Create the controller using the model retreived above. 
            Controller controller = new Controller(model);

            //Start the controller 
            controller.Start();

            //Create parameters for execute. The size of the arrays must match the model size; 
            double[] cvs = new double[] { 0 };
            double[] sps = new double[] { 0 };
            double[] mvs = new double[] { 0 };
            double[] dvs = new double[] { };
            bool run = true; bool quality = false;

            //This will contain the quality after the execute method has been called. 
            int errorId = 0; //This will contain the error id, if any, after the execute method has been called. 
            int iterationToUpdateModel = 10; //The iteration at which an update will occur 
            int iteration = 0; //The current iteration 

            while (run)
            {
                if (iteration == iterationToUpdateModel)
                {
                    //Perform model update 
                    model.QPMaxIterations = 100;
                    model.InputCVs[0].Weight = 0.5;
                    controller.UpdateModel(model);
                }
                //Execute the controller 
                double[] result = controller.Execute(cvs, sps, dvs, mvs, ref quality, ref errorId);
                //Put the thread to sleep for 1 second. This should match the sampling interval. 
                Thread.Sleep(1000); iteration++;
            }
            Console.ReadLine();
        }
    }
}
```
