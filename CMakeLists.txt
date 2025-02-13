# SPDX-License-Identifier: Apache-2.0
# Copyright 2021 IBM Corp.
cmake_minimum_required( VERSION 3.12 )
project( secvarctl C )

set( CMAKE_C_COMPILER gcc )
#sources for secvarctl
set( SRC secvarctl.c generic.c )

#sources for edk2 backend
set ( EDK2SRC edk2-svc-read.c edk2-svc-write.c edk2-svc-validate.c edk2-svc-verify.c edk2-svc-generate.c )
set ( EDK2SRCDIR backends/edk2-compat/ )
list( TRANSFORM EDK2SRC PREPEND ${EDK2SRCDIR} )
list( APPEND SRC ${EDK2SRC} )

#sources for borrowed skiboot code
set ( SKIBOOTSRC secvar_util.c backend/edk2-compat.c backend/edk2-compat-process.c )
set ( SKIBOOTSRCDIR external/skiboot/libstb/secvar/ )
list( TRANSFORM SKIBOOTSRC PREPEND ${SKIBOOTSRCDIR} )
list( APPEND SRC ${SKIBOOTSRC} )
#directory where mbedtls/openssl crypto functions live
set( CRYPTOSRCDIR ${SKIBOOTSRCDIR}/crypto/ )

# include paths:
#  - include/
#  - the root directory so that we can e.g. #include "{external,backends}/..."
#  - skiboot, so we can do <ccan/whatever/whatever.h>
#  - skiboot's include directory, so we can do e.g. <secvar.h>
include_directories(include ${CMAKE_CURRENT_SOURCE_DIR} ${CMAKE_CURRENT_SOURCE_DIR}/external/skiboot
		    ${CMAKE_CURRENT_SOURCE_DIR}/external/skiboot/include )

#User specified options 

#OPENSSL compiles with openssl instead of mbedtls
option( OPENSSL "Compile with OpenSSL as crypto library, default mbedtls" )
#GNUTLS compiles with gnutls instead of mbedtls
option( GNUTLS "Compile with GnuTLS as crypto library, default mbedtls" )
if ( OPENSSL )
  #sources for crypto function implemented w openssl
  set( CRYPTOSRC crypto-openssl.c )
  list( TRANSFORM CRYPTOSRC PREPEND ${CRYPTOSRCDIR} )
elseif( GNUTLS )
   #sources for crypto function implemented w gnutls
   set( CRYPTOSRC crypto-gnutls.c )
   list( TRANSFORM CRYPTOSRC PREPEND ${CRYPTOSRCDIR} )
else ()
  #sources for extra mbedtls functions
  set ( EXTRAMBEDTLSSRC pkcs7_write.c pkcs7.c )
  set ( EXTRAMBEDTLSSRCDIR  external/extraMbedtls/ )
  list( TRANSFORM EXTRAMBEDTLSSRC PREPEND ${EXTRAMBEDTLSSRCDIR} )
  list( APPEND SRC ${EXTRAMBEDTLSSRC} )
  #sources for crypto function implemented w secvarctl
  set( CRYPTOSRC crypto-mbedtls.c )
  list( TRANSFORM CRYPTOSRC PREPEND ${CRYPTOSRCDIR} )
endif()
list ( APPEND SRC ${CRYPTOSRC} )

option( STATIC "Create statically linked executable" OFF )
if ( STATIC )
  set( BUILD_SHARED_LIBRARIES OFF )
  set( CMAKE_EXE_LINKER_FLAGS "-static" )
  set( PTHREAD "pthread" )
endif(  )

#Strip resulting executable for minimal size
option( STRIP "Strip executable of extra data for minimal size" OFF )
if ( STRIP )
  set( BUILD_SHARED_LIBRARIES OFF )
  string( APPEND CMAKE_C_FLAGS "-s" )
endif(  )


add_executable( secvarctl ${SRC} )

#no crypto means don't compile the generate command = smaller executable
option( CRYPTO_READ_ONLY "Build without crypto write functions for smaller executable, some functionality lost" OFF )
if ( NOT CRYPTO_READ_ONLY )
  target_compile_definitions( secvarctl PRIVATE  SECVAR_CRYPTO_WRITE_FUNC )
endif(  )

#append possible extensions for library
LIST( APPEND CMAKE_FIND_LIBRARY_SUFFIXES ".so.0" ".a" ".so" )

#if compiling with openssl, get libraries
if ( OPENSSL )
  find_package( OpenSSL REQUIRED )
  target_link_libraries( secvarctl OpenSSL::SSL )
  target_compile_definitions( secvarctl PRIVATE  SECVAR_CRYPTO_OPENSSL )
elseif ( GNUTLS )
  find_package( GnuTLS REQUIRED )
  target_link_libraries( secvarctl GnuTLS::GnuTLS )
  target_compile_definitions( secvarctl PRIVATE  SECVAR_CRYPTO_GNUTLS )
#else get mbedtls libraries
else()
  #get mbedtls if custom path defined
  if ( DEFINED CUSTOM_MBEDTLS )
      find_library( MBEDX509 mbedx509 PATHS ${CUSTOM_MBEDTLS}/library NO_DEFAULT_PATH REQUIRED )
      find_library( MBEDCRYPTO mbedcrypto PATHS ${CUSTOM_MBEDTLS}/library NO_DEFAULT_PATH REQUIRED )
      find_library( MBEDTLS mbedtls PATHS ${CUSTOM_MBEDTLS}/library NO_DEFAULT_PATH REQUIRED )
      include_directories( ${CUSTOM_MBEDTLS}/include ) 
  else(  )
      find_library( MBEDX509 mbedx509 HINTS ENV PATH REQUIRED )
      find_library( MBEDCRYPTO mbedcrypto HINTS ENV PATH REQUIRED )
      find_library( MBEDTLS mbedtls HINTS ENV PATH REQUIRED )
  endif (  )
  target_link_libraries( secvarctl ${MBEDTLS} ${MBEDX509} ${MBEDCRYPTO} ${PTHREAD} )
  target_compile_definitions( secvarctl PRIVATE  SECVAR_CRYPTO_MBEDTLS )
endif()


#set default build type to release
set( DEFAULT_BUILD_TYPE "Debug" )
if ( NOT CMAKE_BUILD_TYPE )
  set( CMAKE_BUILD_TYPE ${DEFAULT_BUILD_TYPE} )
    message( "Setting build type to default: " ${CMAKE_BUILD_TYPE} )
endif(  )
#allow for different optimizations here
set( CMAKE_C_FLAGS_RELEASE     "-O2 -g" )
set( CMAKE_C_FLAGS_DEBUG       "-O0 -g3 -Wall -Werror" )
set( CMAKE_C_FLAGS_COVERAGE    "-O0 -g3 -fprofile-arcs -ftest-coverage -Wall -Werror" )


#set c standard
set( CMAKE_C_STANDARD 99 )
set( CMAKE_C_STANDARD_REQUIRED ON )

install( FILES ${CMAKE_CURRENT_SOURCE_DIR}/secvarctl.1 DESTINATION ${CMAKE_INSTALL_PREFIX}/share/man/man1 )
install( TARGETS secvarctl DESTINATION bin )
