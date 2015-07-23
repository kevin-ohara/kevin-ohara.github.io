---
layout: post
title:  "Delete a Remote File with SFTP and Spring Integrations"
comments: true
date:   2015-07-09 10:24:39
categories: code
---

I've recently been working with Spring specifically Spring Batch.  I've always thought of Spring as a huge behemoth of a framework and it really isn't.  If used properly it can be relatively lightweight however it is often misused by developers, my most feared Git comment these days is 'added Spring for DI' but that's a story for another day.

Anyway the problem that I faced was I needed to delete any existing files that matched a certain criteria from a remote directory.  Sounds simple right?  In hindsight it is relatively simple but I had little to no knowledge of Spring and so it would be a process of discovery for me and took a bit longer than expected.  I didn't find any real concrete examples of what I wanted to achieve specifically and so I was relying heavily on the official documentation, which is very good but at times needs work :S

First up in Spring Batch you configure your jobs in steps using XML and link it to a bean which is attached to our tasklet

```xml
<step id="cleanSFTPDirectory">
    <tasklet ref="cleanSFTPDirectoryTasklet">
        <listeners>
            <listener ref="myStepListener" />
        </listeners>
    </tasklet>
    <next on="COMPLETED" to="sendFilesToSFTP" />
</step>
```

In the bean configuration we can inject in the properties that we will require in the tasklet, here we need our channel, the file pattern to be deleted and target directory on the SFTP server.

```xml
<bean id="cleanSFTPDirectoryTasklet" class="com.example.integration.cleanSFTPDirectoryTasklet" scope="step">
    <property name="channel" ref="rmChannel" />
    <property name="filePatternToDelete" value="${sftp.fileNameDeletePattern}" />
    <property name="targetDirectory" value="${sftp.remoteDirectory}/"/>
</bean>
```

If you are wondering what a channel is, it really does what it says on the box; it just allows producers to pass Messages to consumers or vice versa but the important thing is that it decouples the producer from the consumer.  Here is the config for the SFTP (you will need standard set up too e.g. xmls:int/xmls:int-sftp etc.)

```xml
<bean id="sftpSessionFactory"
    class="org.springframework.integration.sftp.session.DefaultSftpSessionFactory">
    <property name="host" value="${sftp.host}" />
    <property name="port" value="${sftp.port}" />
    <property name="user" value="${sftp.user}" />
    <property name="privateKey" value="file:${sftp.privateKey}" />
    <property name="password" value="${sftp.password}" />
</bean>

<int:channel id="rmChannel"/>

<int-sftp:outbound-gateway
        session-factory="sftpSessionFactory"
        request-channel="rmChannel"
        reply-channel="nullChannel"
        remote-file-separator="/"
        command="rm"
        expression="headers['file_remoteDirectory'] + headers['file_remoteFile']" />
```

Okay so some important stuff is happening here, first we create our sftpSessionFactory and then define our channel `rmChannel` that we used in the previous bean to inject into our tasklet.  The most interesting thing here is the outbound-gateway, to understand this you need to understand [Spring SFTP Adapters] provided by Spring Integrations.  There's a lot of reading but essentially there are 3 main adapters:

*Inbound Channel Adapters* these are essentially a listener that are bound to a remote directory, they will wait for an even e.g. a new file and initiate a transfer of that file to specified location.

*Outbound Channel Adapters* these take a payload from a message, usually a file, connect to a remote directory and transfer the file there.

*Outbound Gateway* these provide a limited set of commands that allow you to interact on with a remote SFTP server.  This is what I was interested in, the commands are:

* ls
* get
* mget
* rm
* mv
* put
* mput

Chaining these commands can give you some really nice process flows as you can pass results via reply-channels to other gateways.  You will notice in my outbound-gateway configuration I've set my reply-channel to `nullChannel` as I don't care about the result, I just want to send a remove command.  We also need to specify our remote direcroy and remote file via the headers `file_remoteDirectory` and `file_remoteFile`.  Make sure to have a trailing `/` in your directory, I got stung with this and took me a while to figure out why it couldn't find the file to delete!

So now we can tie all this together and kick off the process in our tasklet

```java
public class cleanSFTPDirectoryTasklet implements Tasklet, InitializingBean {

	private MessageChannel channel;
	private String filePatternToDelete;
	private String targetDirectory;

	@Override
	public void afterPropertiesSet() throws Exception
	{
		Assert.notNull(channel, "channel cannot be null");
		Assert.notNull(filePatternToDelete, "filePatternToDelete cannot be null");
		Assert.notNull(targetDirectory, "targetDirectory cannot be null");
	}

	@Override
	public RepeatStatus execute(StepContribution stepContribution, ChunkContext chunk) throws Exception
	{
		Message<Boolean> deleteRequest = MessageBuilder.withPayload(true)
				.setHeader("file_remoteDirectory", targetDirectory)
				.setHeader("file_remoteFile", filePatternToDelete)
				.build();

		channel.send(deleteRequest);

		return RepeatStatus.FINISHED;
	}

    // setters
}
```

Hopefully someone finds this of some help.

[Spring SFTP Adapters]: http://docs.spring.io/spring-integration/docs/latest-ga/reference/html/sftp.html
