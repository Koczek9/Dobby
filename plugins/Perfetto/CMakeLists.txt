# If not stated otherwise in this file or this component's LICENSE file the
# following copyright and licenses apply:
#
# Copyright 2020 Sky UK
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

add_library( PerfettoPlugin
        MODULE
        source/PerfettoPlugin.cpp
        $<TARGET_OBJECTS:DobbyPluginCommonLib>
        )

target_include_directories( PerfettoPlugin
        PRIVATE
        $<TARGET_PROPERTY:DobbyPluginCommonLib,INTERFACE_INCLUDE_DIRECTORIES>
        $<TARGET_PROPERTY:DobbyDaemonLib,INTERFACE_INCLUDE_DIRECTORIES>
        $<TARGET_PROPERTY:AppInfraLogging,INTERFACE_INCLUDE_DIRECTORIES>
        )

set_target_properties( PerfettoPlugin PROPERTIES
        VERSION   ${DOBBY_MAJOR_VERSION}.1.0
        SOVERSION ${DOBBY_MAJOR_VERSION}
        )

install(
        TARGETS PerfettoPlugin
        LIBRARY DESTINATION lib/plugins/dobby
)
