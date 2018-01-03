# chat
<property name="failure.reason" value="svn-add" />
		 <exec executable="perl" failonerror="false" resultproperty="svn-add.rc">
		        <arg value="${env.HUDSON_HOME}/admin/shared/runsvn.pl" />
			<arg value="add" />
			<arg value="${addDir}" />
		</exec>
