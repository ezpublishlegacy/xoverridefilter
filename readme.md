X Override Filter
=================

Introduction
-----------
With this simple extension, you can define your business logic(a php class) with template override in override.ini, so business logic can be **easily and clearly** done in a pure php class instead of custom operator, complicated templating, or datatype.

The initial idea came from pull request: https://github.com/ezsystems/ezpublish-legacy/pull/694 which is in eZ Publish 5.2 automatically.

Small example:

Definition in override.ini

    [myform_view]
    Source=node/view/full.tpl
    MatchFile=myform.tpl
    Subdir=templates
    Match[class_identifier]=myform
    #Class is new :)
    Class=myFormView
Implement template form.tpl

        {if is_set( $result )}
        <div>{$result}</div>
        {/if}
        <form action="" method="post">
            <div>
                Name: <input type="text" name="name" value="" />
            </div>
            <div>
                Email: <input type="text" name="email" value="" />
            </div>
            <div>
                <input type="submit" name ="SubmitButton" value='Submit' />
                <input type="submit" name ="DiscardButton" value='Discard' />
            </div>
        </form>


Implement class myFormView

    class myFormView implements xNodeviewRender
    {
      function initNodeview( $module, $node, $tpl, $viewMode )
      {
          $tpl->setVariable( 'cache_ttl', 0 );

          $http = eZHTTPTool::instance();
          if( $http->hasVariable( 'SubmitButton' ) )
          {
             // Show result if the form submit
             $name = $http->variable( 'name' );
             $email = $http->variable( 'email' );
             $tpl->setVariable( 'result', ezpI18n::tr( 'example', "You inputed Name: %1, Email %2", '', array( $name, $email )  ) );
          }
          else if( $http->hasVariable( 'DiscardButton' ) )
          {
             // Redirect to homepage if the form is discarded.
             $module->redirectTo( '/' );
          }
      }
    }

Enhanced override.ini
---------------------
Enhanced override.ini supports

1. Class tag to identify a php class implenetation of the logic.
2. Match[attribute_\<attribute_identifier\>]=\<value\> to better filter template.
3. Match[node], Match[class_identifer], Match[viewmode] for view logic conditions

The 3 above can be combined with existing template override.

Requirements
---------

- For eZ Publish 5.2: it works well

- For eZ Publish 4.2 - 5.1: it needs a kernel patch(not an unauthorized hack, but a feature backport). see doc/patches/event-pre_rending-*.diff


Install
--------
1. Copy this extension under \<ezp_root\>/extension
2. If your eZ Publish is \<5.2 (e.g. 4.7), run these commands under \<ezp_root\>
   
         cp extension/xoverridefilter/doc/patches/event-pre_rending-4.5-4.7.diff ./
         patch -p0 < event-pre_rending-4.5-4.7.diff --dry-run
         patch -p0 < event-pre_rending-4.5-4.7.diff
       
    P.S. For community versioning, 4.5-4.7 = community 2011.5-2012.5 
2. Activate extension xoverridefilter
3. Clear ini cache
4. Regenerate autoload array


Example(See doc/example for code)
---------


1. Configure condition and class under myextension

   **Scenario 1 - Custom view logic** 
  
   extension/myextension/settings/override.ini.append.php.

        [myform_view]
        Match[class_identifier]=myform
        Class=myFormView
     
   The configuration above means that ‘myform’ objects will use myFormView for view logic. Form templates can be defined in additional template override rules.

   **Scenario 2 - Custom view logic with custom template**. You can also combine view logic with template override in one override rule. 

        [myform_view_2]
        Source=node/view/full.tpl
        MatchFile=form.tpl
        Subdir=templates
        Match[class_identifier]=myform
        #Condition section_identifier will be ignored by custom view logic.
        Match[section_identifier]=standard
        Class=myFormView

   The configuration above means that, 'myform' objects under Standard section will use class myFormView as view logic and form.tpl as template; while 'myform' objects under other sections will use myFormView as view logic and full.tpl(if no other override rule applies) as template.

**Scenario 3 - Use attribute for override match. It's better to use this instead of node_id**. For example:

      [article_breaking-news_full]
      Source=node/view/full.tpl
      MatchFile=break-news.tpl
      Subdir=templates
      Match[attribute_article_identifier]=breaking-news
      Match[class_identifier]=article
    
      [article_campaign_full]
      Source=node/view/full.tpl
      MatchFile=campaign.tpl
      Subdir=templates
      Match[attribute_article_identifier]=campaign
      Match[class_identifier]=article

2. Implement template form.tpl

    extension/myextension/design/standard/override/templates/form.tpl

        {if is_set( $result )}
        <div>{$result}</div>
        {/if}
        <form action="" method="post">
            <div>
                {"Name:"|i8n('example')} <input type="text" name="name" value="" />
            </div>
            <div>
                {"Email:"|i18n('example')} <input type="text" name="email" value="" />
            </div>
            <div>
                <input type="submit" name ="SubmitButton" value={'Submit'|i18n( 'example' )} />
                <input type="submit" name ="DiscardButton" value={'Discard'|i18n( 'example' )} />
            </div>
        </form>

3. Implement class myFormView

    extension/myextension/classes/myformview.php

        <?php
        class myFormView implements xNodeviewRender
        {
         /**
          * This method is invoked before template is fetched.
          *
          * Typical usage:
          * 1. Set php variable to template directly instead of use eZ custom operator in template
          * 2. Customize http form action
          *
          */
          public function initNodeview( $module, $node, $tpl, $viewMode )
          {
               // Disable view cache for this page, since the form will be dynamic
               $tpl->setVariable( 'cache_ttl', 0 );

               $http = eZHTTPTool::instance();
               if( $http->hasVariable( 'SubmitButton' ) )
               {
                    // Show result if the form submit
                    $name = $http->variable( 'name' );
                    $email = $http->variable( 'email' );
                    $tpl->setVariable( 'result', ezpI18n::tr( 'example', "You inputed Name: %1, Email %2", '', array( $name, $email )  ) );
               }
                else if( $http->hasVariable( 'DiscardButton' ) )
                {
                    // Redirect to homepage if the form is discarded.
                    $module->redirectTo( '/' );
                }

          }
        }
        ?>

4. Regenerated autoload array for extension
<php path> bin/php/ezpgenerateautoloads.php -e

5. Clear cache before viewing the page(content/view/full/50).

FAQ
---------
1. Is the business logic inside view cache?

   Yes, for first visit eZ will invoke the business logic and generate view cache, for second visit it may skip business logic and load view cache only if the view cache keys are not changed.

   Because of that, for high dynamic page(like form), it's recommanded to disable view cache from php: 
   
         public function initNodeview( $module, $node, $tpl, $viewMode )
         {
             $tpl->setVariable( 'cache_ttl', 0 );


Roadmap
--------
Should

- Support override rules hierarchy

Could

- Control cache via persistant variable/cache block keys


Feedback?
----------
Welcome to give any comments/suggestions. 

Review it on projects.ez.no: http://projects.ez.no/xoverridefilter/reviews .

Contact me? You can reach me by share.ez.no: http://share.ez.no/community/profile/86893

Issue tracker: https://github.com/xc/xoverridefilter/issues


