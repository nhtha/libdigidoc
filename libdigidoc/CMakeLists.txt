configure_file(
	${CMAKE_SOURCE_DIR}/etc/digidoc.conf.cmake
	${CMAKE_CURRENT_BINARY_DIR}/${DIGIDOC_CONF_NAME}
	@ONLY
)
FILE( GLOB CERTS ${CMAKE_SOURCE_DIR}/etc/certs/*.crt )

set( PUBLIC_HEADER
	DigiDocCert.h
	DigiDocConfig.h
	DigiDocConvert.h
	DigiDocDebug.h
	DigiDocDefs.h
	DigiDocDfExtract.h
	DigiDocEncGen.h
	DigiDocEnc.h
	DigiDocEncSAXParser.h
	DigiDocError.h
	DigiDocGen.h
	DigiDocLib.h
	DigiDocMem.h
	DigiDocObj.h
	DigiDocOCSP.h
	DigiDocParser.h
#	DigiDocPKCS11.h
	DigiDocSAXParser.h
	DigiDocStack.h
	DigiDocVerify.h
	DigiDocHTTP.h
	DigiDocService.h
)

if( WIN32 )
	add_definitions(
		-DWITH_SOAPDEFS_H
		-D_CRT_NONSTDC_NO_DEPRECATE
		-D_CRT_SECURE_NO_DEPRECATE
		-D_CRT_SECURE_NO_WARNINGS
		-D_SCL_SECURE_NO_WARNINGS
	)
	list( APPEND libdigidoc_SRCS DigiDocGlobals.c DigiDocCSP.c DigiCrypt.c DlgUnit.c DlgUnitS.c )
	set( EXT_LIBRARIES Crypt32 Comctl32 )
endif()

add_library( digidoc ${BUILD_TYPE}
	${PUBLIC_HEADER}
	${CMAKE_CURRENT_BINARY_DIR}/${DIGIDOC_CONF_NAME}
	${libdigidoc_SRCS}
	${CERTS}
	libdigidoc.rc
	DigiDocConfig.c
	DigiDocLib.c
	DigiDocObj.c
	DigiDocPKCS11.c
	DigiDocError.c
	DigiDocParser.c
	DigiDocDebug.c
	DigiDocSAXParser.c
	DigiDocMem.c
	DigiDocStack.c
	DigiDocEnc.c
	DigiDocEncGen.c
	DigiDocEncSAXParser.c
	DigiDocCert.c
	DigiDocConvert.c
	DigiDocGen.c
	DigiDocVerify.c
	DigiDocOCSP.c
	DigiDocDfExtract.c
	DigiDocHTTP.c
	DigiDocService.c
)

target_link_libraries( digidoc
	${CMAKE_DL_LIBS}
	${LIBXML2_LIBRARIES}
	${OPENSSL_LIBRARIES}
	${ZLIB_LIBRARIES}
	${EXT_LIBRARIES}
)

set_target_properties( digidoc PROPERTIES
	VERSION ${MAJOR_VER}.${MINOR_VER}.${RELEASE_VER}
	SOVERSION 2
	PUBLIC_HEADER "${PUBLIC_HEADER}"
	RESOURCE ${CMAKE_CURRENT_BINARY_DIR}/${DIGIDOC_CONF_NAME}
	FRAMEWORK_VERSION 2
	FRAMEWORK "${FRAMEWORK}"
	MACOSX_FRAMEWORK_IDENTIFIER "ee.ria.libdigidoc"
	MACOSX_RPATH YES
)

if( BUILD_TOOLS )
	add_executable(cdigidoc cdigidoc.c cdigidoc.rc)
	target_link_libraries(cdigidoc digidoc)
endif()

install( TARGETS digidoc
	RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
	LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
	ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
	RESOURCE DESTINATION ${CMAKE_INSTALL_SYSCONFDIR}
	PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/libdigidoc
	FRAMEWORK DESTINATION /Library/Frameworks
)

if(WIN32)
	install(FILES $<TARGET_PDB_FILE:digidoc> $<TARGET_PDB_FILE:cdigidoc> DESTINATION ${CMAKE_INSTALL_BINDIR} OPTIONAL)
endif()

if( FRAMEWORK )
	set_target_properties( digidoc PROPERTIES
		OUTPUT_NAME "libdigidoc"
		COMPILE_DEFINITIONS "FRAMEWORK"
		LINK_FLAGS "-framework CoreFoundation -framework Security"
		RESOURCE "${CERTS};${CMAKE_CURRENT_BINARY_DIR}/${DIGIDOC_CONF_NAME}"
	)
	add_custom_command( TARGET cdigidoc POST_BUILD
		COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:cdigidoc> $<TARGET_FILE_DIR:digidoc>/Resources )
	add_custom_target( codesign DEPENDS cdigidoc
		COMMAND codesign -f -s ${SIGNCERT}
			$<TARGET_FILE_DIR:digidoc>/Resources/cdigidoc
			$<TARGET_FILE_DIR:digidoc>/../..
		COMMAND touch $<TARGET_FILE:cdigidoc>
	)
	if(INSTCERT)
		set(PKGEXTRA --sign ${INSTCERT})
	endif()
	add_custom_target( pkgbuild DEPENDS cdigidoc
		COMMAND make install DESTDIR=install
		COMMAND pkgbuild --root install ${PKGEXTRA}
			${CMAKE_BINARY_DIR}/libdigidoc_${VERSION}$ENV{VER_SUFFIX}.pkg
	)
	add_custom_target( zipdebug DEPENDS cdigidoc
		COMMAND dsymutil -o libdigidoc.dSYM $<TARGET_FILE:digidoc>
		COMMAND dsymutil -o libdigidoc.dSYM $<TARGET_FILE:cdigidoc>
		COMMAND zip -r ${CMAKE_BINARY_DIR}/libdigidoc-dbg_${VERSION}$ENV{VER_SUFFIX}.zip libdigidoc.dSYM
	)
else()
	configure_file( libdigidoc.pc.cmake libdigidoc.pc @ONLY )
	install( FILES ${CMAKE_CURRENT_BINARY_DIR}/libdigidoc.pc DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig )
	install( FILES ${CERTS} DESTINATION ${CMAKE_INSTALL_FULL_DATADIR}/libdigidoc )
	if( BUILD_TOOLS )
		install( TARGETS cdigidoc DESTINATION ${CMAKE_INSTALL_BINDIR} )
		configure_file( cdigidoc.1.cmake cdigidoc.1 )
		install( FILES ${CMAKE_CURRENT_BINARY_DIR}/cdigidoc.1 DESTINATION ${CMAKE_INSTALL_MANDIR}/man1 )
	endif()
endif()

#install( FILES
#	pkcs11/pkcs11.h
#	pkcs11/pkcs11f.h
#	pkcs11/pkcs11t.h
#	pkcs11/unix.h
#	pkcs11/cryptoki.h
#	DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/libdigidoc/pkcs11
#)
