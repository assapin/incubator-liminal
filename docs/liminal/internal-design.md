<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# Liminal Internal design

This document describes the internal mechanisms used by Apache Liminal to turn your Liminal YAML files
into running Airflow DAGs.


## Concepts

**Task** - the unit of processing logic in Apache Liminal, for example a Python script or a PySpark program.
A Task has an entrypoint such as a main() function, and that entrypoint may take parameters - which are defined in the
YAML file as *Variables*.

A *Task* has a type (such as a Python Task, a PySpark task etc.).
The type determines how the task gets executed by an *Executor*. 

**Executor** - in charge of executing tasks.
Executors are able to interpret *Tasks* and send them to be executed on some target platform.
Built in executors include *K8s executor* and *EMR executor*, that are able to send tasks to be executed on a K8S cluster,
and on an EMR cluster accordingly.

However, before tasks can be sent to run on e.g. K8s, they need to by **built**

**Builder** - For a *Task* to be executed on it's target platform, the user provided code needs to be built into a packaged artefact.
For example, to run a *Task* on K8s, the user code needs to be built into a docker image (and then pushed into a 
Container registry that is available for the cluster).

The role of the *Builder* is to build user code into packaged artefacts.
Liminal comes provided with Builders that can build Docker images from Tasks and Services.
There are several variants of the Image builder, whith different base `Dockerfile`s that determine the image layout.

## The lifecycle of a Liminal Pipeline

![](../nstatic/liminal_hl-flow.png)

**1** : `liminal build` runs the *Builders* on the user's YAML and Python files, and creates a Docker container.

**2** :  `liminal create` scans the YAML files and detects any PersistentVolumes that need to be created, then creates them
**3** : `liminal deploy` picks up the YAML file and places it in a folder that Airflow uses to periodically scan for DAGs
**4** : Airflow scheduler scans the DAGs folder, and encounters a Liminal python file that was pre-installed with Airflow.
    This Python file acts as a DAG factory - i.e. it scans the DAGs folder for any Liminal YAML files, and traslates them    
    into Airflow DAGs. For example - Tasks that were deinfed in the YAML get trasnlated into Airflow Task object, and Executores
    get translated into Airflow Operators. 
**5** : The returned DAG is registered with Airflow
**6** : Airflow scheduler starts the DAG, and the Operator executes a Task that runs on Kuberentes
**7** : The User's Docker image from step (1) is executed as a Pod on Kuberentes, using the mounted volume created in step (2)





