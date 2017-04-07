---
layout: post
title: "Velocity_Html_Escape"
date: 2017-04-07 23:03:22 +0800
comments: true
categories: 
---
##
> 最近在做发邮件相关的工作，需要用到Velocity。个人velocity来说就只是知道这个是个模版框架而已。这个功能给QA测过后有一个bug是关于html没有
被escape。这篇文章就是记录今天改这个bug的一个过程。
###增加Html escape的支持
项目中通过配置Spring中VelocityEngineFactoryBean以达到对Velocity的支持。

	<bean id="velocityEngineBean" class="org.springframework.ui.velocity.VelocityEngineFactoryBean" scope="singleton">
		<property name="resourceLoaderPath" value="classpath:"/>
		<property name="preferFileSystemAccess" value="true"/>
		<!-- this method of using commons logging is deprecated, so set to false -->
		<property name="overrideLogging" value="false"/>
		<property name="velocityProperties">
			<value><!-- setup classpath resource loader -->
				resource.loader=classpath
				classpath.resource.loader.class=org.apache.velocity.runtime.resource.loader.ClasspathResourceLoader
				classpath.resource.loader.cache=true
				classpath.resource.loader.modificationCheckInterval=-1
				userdirective=com.active.services.endurance.commons.velocity.I18nDirective
				<!-- setup slf4JLogChute so velocity logs into slf4j logs -->
				runtime.log.logsystem.class=com.active.services.core.util.Slf4jLogChute
				runtime.log.logsystem.log4j.logger=org.apache.velocity.app.VelocityEngine
				eventhandler.referenceinsertion.class = org.apache.velocity.app.event.implement.EscapeHtmlReference
      </value>
		</property>
	</bean>
	
通过诸如velocityEngineBean，然后通过VelocityEngineUtils将vm模版merge成最后的html。

`VelocityEngineUtils.mergeTemplateIntoString(getVelocityEngine(), formattedLocale, "UTF-8", this.getEmailModel());`

以下部分配置就是对Html Escape的支持

`eventhandler.referenceinsertion.class = org.apache.velocity.app.event.implement.EscapeHtmlReference`

通过增加

`eventhandler.escape.html.matc = abc*`

可以达到只escape以abc开始的变量值比如**${abc123}**
##UserDirective带来的困扰
通过以上配置可以使Velocity在merge template的时候将变量值escape html tag。但是项目里面还使用了*userdirective=com.active.services.endurance.commons.velocity.I18nDirective*
用于对i18n的支持。

`<td class="column" width="600" colspan="2" align="left" valign="top" style="padding: 30px 30px 10px 30px; font-family: arial,sans-serif; font-size: 16px; line-height: 18px; color: #555;">
             <p>
                 #i18n("racepass.confirmation.emal.greeting", [${participantName}])
             </p>
         </td>`
         
*racepass.confirmation.emal.greeting*就是i18n的key，而*[${participantName}]*这里面的变量用于替换i18n值里面的变量。比如
*racepass.confirmation.emal.greeting=Dear {0}*

处理这个转换逻辑的代码就在I18nDirective里面



    @SuppressWarnings("unchecked")
     @Override
     public boolean render(InternalContextAdapter context, Writer writer, Node node) throws IOException, ResourceNotFoundException,
             ParseErrorException, MethodInvocationException {
         MessageSource messageSource = (MessageSource) context.get("_messageSource");
         Locale locale = (Locale) context.get("_locale");
         String msgKey = valueOfNode(context, node.jjtGetChild(0));         
         String[] msgParams = null;
         if (node.jjtGetNumChildren() == 2) {
             Object o = valueOfNode(context, node.jjtGetChild(1));
             msgParams = o instanceof List ? ((List<String>) o).toArray(new String[0]) : (String[]) o;
         }
         String localizedMsg = messageSource.getMessage(msgKey, msgParams, locale);
         writer.write(localizedMsg);
         return true;
     }
   
     
      
      @SuppressWarnings("unchecked")
      private <T> T valueOfNode(InternalContextAdapter context, Node node) {
          return (T) node.value(context);
      }
  
  
问题就出来通过node.value直接取值是不会经过eventhandler的，通过简单debug对比发现可以通过node.render方法就完美兼容eventhandler,遂增加下面方法取值
    
    private <T> T valueOfNodeProcessedByEventHandler(InternalContextAdapter context, Node node) throws IOException {
         if (node instanceof ASTObjectArray) {
             ArrayList results = new ArrayList();
             int size = node.jjtGetNumChildren();
             for (int i = 0; i < size; i++) {
                 Node child = node.jjtGetChild(i);
                 StringWriter writer = new StringWriter();
                 child.render(context, writer);
                 results.add(writer.getBuffer().toString());
             }
             return (T) results;
         }
         return (T) node.value(context);
     }
     