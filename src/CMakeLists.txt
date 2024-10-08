# Copyright 2010, Thomas Moulard, LAAS-CNRS
#
# This file is part of roboptim-core.
# roboptim-core is free software: you can redistribute it and/or modify
# it under the terms of the GNU Lesser General Public License as published by
# the Free Software Foundation, either version 3 of the License, or
# (at your option) any later version.
#
# roboptim-core is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Lesser Public License for more details.
# You should have received a copy of the GNU Lesser General Public License
# along with roboptim-core.  If not, see <http://www.gnu.org/licenses/>.

# Define the directory where plug-ins will be installed.
SET(PLUGINDIR ${CMAKE_INSTALL_LIBDIR}/${ROBOPTIM_CORE_RELPLUGINDIR})
MESSAGE("-- Plug-in directory: ${PLUGINDIR}")

MACRO(IPOPT_PLUGIN NAME)
  ADD_LIBRARY(roboptim-core-plugin-${NAME} MODULE
    ${NAME}.cc tnlp.cc tnlp.hh tnlp.hxx doc.hh ${HEADERS}
    )

  IF(APPLE)
    SET(CMAKE_SHARED_MODULE_CREATE_CXX_FLAGS "${CMAKE_SHARED_LIBRARY_CREATE_CXX_FLAGS} -undefined dynamic_lookup")
  ENDIF()

  SET_TARGET_PROPERTIES(roboptim-core-plugin-${NAME} PROPERTIES
    COMPILE_DEFINITIONS
      IPOPT_DEFAULT_LINEAR_SOLVER="${IPOPT_DEFAULT_LINEAR_SOLVER}"
    COMPILE_FLAGS "-Droboptim_core_plugin_ipopt_EXPORTS"
    PREFIX "")
  IF(NOT APPLE)
    SET_TARGET_PROPERTIES(roboptim-core-plugin-${NAME}
      PROPERTIES VERSION 3.2.0 SOVERSION 3)
  ENDIF()
  INSTALL(TARGETS roboptim-core-plugin-${NAME}
    EXPORT ${TARGETS_EXPORT_NAME}
    DESTINATION ${PLUGINDIR})
  PKG_CONFIG_USE_DEPENDENCY(roboptim-core-plugin-${NAME} ipopt)
  #PKG_CONFIG_USE_COMPILE_DEPENDENCY(roboptim-core-plugin-${NAME} roboptim-core)
  TARGET_LINK_LIBRARIES(roboptim-core-plugin-${NAME} roboptim-core::roboptim-core Eigen3::Eigen)
  
  # Make sure all symbols are defined.
  # See: http://bugs.debian.org/cgi-bin/bugreport.cgi?bug=592752
  IF(CMAKE_SYSTEM_NAME MATCHES "Linux"
    AND EXISTS "/etc/debian_version"
    AND IPOPT_PREFIX MATCHES "/usr")
    TARGET_LINK_LIBRARIES(roboptim-core-plugin-${NAME} dmumps_seq)
  ENDIF()
ENDMACRO()

# Ipopt plug-in (with only gradient computation).
IPOPT_PLUGIN(ipopt)
# Ipopt plug-in (with only gradient computation and sparse matrices).
IPOPT_PLUGIN(ipopt-sparse)
# Ipopt twice-differentiable (td) plug-in (requires hessian computation).
IPOPT_PLUGIN(ipopt-td)
