(function ($) {
  'use strict';

  Drupal.behaviors.euccx = {
    attach: function (context, settings) {

      $(window).load(function () {
        Drupal.euccx.init();
      });

    }
  };

  Drupal.euccx = {};

  // Entry function for our script.
  // Delegates the jobs to other functions.
  Drupal.euccx.init = function () {
    // Check if our environment is valid and return if it's not
    if (!Drupal.euccx.verifyEnvironment()) {
      return;
    }
    Drupal.euccx.handleBanner();
    Drupal.euccx.handleCookieSettingsPage();
    Drupal.euccx.processStatus(Drupal.euccx.getCookieStatus());
  };

  // Since we are using the state of EU Cookie Compliance in multiple steps of
  // the process, we need to make sure that its main function is available.
  // This is important to avoid javascript errors in administrative pages where
  // the euccx block is loaded but the EUCC is "out of context".
  Drupal.euccx.verifyEnvironment = function () {
    return (typeof Drupal.eu_cookie_compliance !== 'undefined' &&
      Drupal.eu_cookie_compliance !== null);
  };

  /**
   * Helper function which returns true, if eucc uses opt-in by categories
   * method. This has several larger implications for us.
   *
   * See https://www.drupal.org/project/euccx/issues/3144073 for details.
   *
   * @return boolean
   */
  Drupal.euccx.usesCategories = function() {
    return Drupal.settings.eu_cookie_compliance.method === 'categories';
  }

  // Handle clicks on the main banner
  Drupal.euccx.handleBanner = function () {
    $(document).ready(function () {
      // When the user clicks on the agree button of the EU Cookie Compliance
      // module we consider it a "carte blanche" consent for all cookies that
      // are defined by our plugins. Instead of waiting for the next page
      // refresh, we can attach an event that unblocks everything right here,
      // right now.
      $('.agree-button').click(function () {
        Drupal.euccx.unblockAll();
        Drupal.euccx.yettUnblockAll();
      });

      // If the administrator did not enable the option in the admin interface
      // we do not need to assign any kind of actions to elements that have the
      // class "euccx-disable-all".
      if (!Drupal.settings.euccx.dab) {
        return;
      }
      // When the user clicks on the disagree button, all we need to do is set
      // our cookie to a blank value (by convention). This will tell prevent our
      // javascript from enabling anything that was blocked in the beginning.
      $('body').on('click', '.euccx-disable-all', function (e) {
        e.preventDefault();
        Drupal.euccx.setCookie('0');
        // We also need to call the decline action of the EU Cookie Compliance
        // module which will handle the status of EUCC and also close the banner
        // or do some other stuff depending on its settings.
        Drupal.eu_cookie_compliance.declineAction();
      });
    });
  };

  // If we are in the page where the cookie settings' block is included, we need
  // to do some stuff.
  Drupal.euccx.handleCookieSettingsPage = function () {
    var cookieBlock = $('#cookie-tabs');
    // If the cookie management block is not in the page, exit.
    // No need to run extra code for something that is not even in the page
    if (!cookieBlock.length) {
      return;
    }

    // Create the jquery ui tabs for the cookie management block
    cookieBlock.tabs();

    // Behaviour is different when using opt-in categories.
    if (!Drupal.euccx.usesCategories()) {
      // When opt-in categories are NOT used, we are offering switches
      // for each script.
      $('#sliding-popup').hide();
      // ... handle the switches
      Drupal.euccx.handleSwitches();
      // ... and attach functionality for the save key
      Drupal.euccx.handleCookieSettingsBehaviors();
    } else {
      // Special handling for category switches:
      Drupal.euccx.handleCategorySwitches();
    }
  };

  // The function that decides what to do with the current user's status.
  // Responsible for unblocking javascript when needed.
  Drupal.euccx.processStatus = function (status) {

    // If the main eucc cookie is not set, it means that the user has not agreed
    // to anything yet. Move on.
    // We also check if the status of cookie-eucc is equal to the string "0"
    // which means that the user denied the cookies.
    if (!status['cookie-eucc'] || status['cookie-eucc'] === '0') {
      // If the main cookie has not been set yet, we have no reason to override
      // the blocks yet. This way, if for example the user accepts all our
      // cookies we will not miss on a google adsense load.
      return;
    }

    // If the euccx cookie was never set but the eucc cookie was set (checked
    // for that above) it means that the user agreed to everything so we need to
    // unblock everything and move on.
    if (!status['cookie-euccx'] && !Drupal.euccx.usesCategories()) {
      Drupal.euccx.unblockAll();
      Drupal.euccx.yettUnblockAll();
      return;
    }

    // We are now in the case where both cookies are set. We need to surgically
    // unblock code depending on its plugin's settings.
    // We use getAcceptedCategoriesPluginNames() to respect opt-in categories
    // if used.
    var pluginNames = Drupal.euccx.getAcceptedCategoriesPluginNames();
    var settings = Drupal.settings.euccx;
    for (var i = 0; i < pluginNames.length; i++) {
      var currentPlugin = pluginNames[i];
      // If current plugin is not in the list of the plugins that the user chose
      // to keep enabled, we move on to the next plugin.
      if (status['cookie-euccx'] && status['cookie-euccx'].indexOf(currentPlugin) === -1) {
        Drupal.euccx.handleBlockOverrides(currentPlugin);
        continue;
      }


      // If the specific plugin has a YETT blacklist defined, we need to get it
      // and whitelist the set of rules
      var blacklist = false;
      if (settings['plugins'] && settings['plugins'][currentPlugin] &&
        settings['plugins'][currentPlugin]['blacklist']) {
        blacklist = settings['plugins'][currentPlugin]['blacklist'];
      }

      if (blacklist) {
        Drupal.euccx.yettUnblockPlugin(blacklist);
      }

      // If the specific plugin has a js_exclude list, there will be a function
      // defined that we need to call in order to run the scripts.
      var js_exclude = false;
      if (settings['plugins'] && settings['plugins'][currentPlugin] &&
        settings['plugins'][currentPlugin]['js_exclude']) {
        js_exclude = settings['plugins'][currentPlugin]['js_exclude'];
      }

      // Unblock if js_exclude OR no specific cookie has been set
      // (which can only happen if using opt-in categories)
      if (js_exclude || !status['cookie-euccx']) {
        Drupal.euccx.unblockPlugin(currentPlugin);
      }

    }
  };

  // Check whether our plugin has defined any block overrides. If it did, watch
  // the DOM for the selector and act accordingly.
  Drupal.euccx.handleBlockOverrides = function (plugin) {
    $(document).ready(function () {
      var settings = Drupal.settings.euccx;
      if (settings && settings['plugins']) {
        var plugins = [];
        if (plugin === '') {
          for (var pluginName in settings['plugins']) {
            if (!{}.hasOwnProperty.call(settings.plugins, pluginName)) {
              continue;
            }
            plugins.push(pluginName);
          }
        }
        else {
          if ({}.hasOwnProperty.call(settings.plugins, plugin)) {
            plugins.push(plugin);
          }
        }
        for (var i = 0; i < plugins.length; i++) {
          if (!{}.hasOwnProperty.call(
            settings['plugins'][plugins[i]],
            'overrides')) {
            continue;
          }
          for (var key in settings['plugins'][plugins[i]]['overrides']) {
            if (!{}.hasOwnProperty.call(
              settings['plugins'][plugins[i]]['overrides'], key)) {
              continue;
            }
            $(key)
              .empty()
              .append(settings['plugins'][plugins[i]]['overrides'][key]);
          }
        }
      }
    });
  };

  // Make sure that when the user hits the save button, it actually does
  // something.
  Drupal.euccx.handleCookieSettingsBehaviors = function () {
    $('body').on('click', '.cookie-settings-save', function (e) {
      try {
        e.preventDefault();
        // Make sure that the EU Cookie Compliance Module has set its cookie
        Drupal.eu_cookie_compliance.acceptAction();
        var plugin;
        // Create our cookie content
        // We need to at least have some content in the cookie in order to make
        // sure that we can check its contents easier later (if for example the
        // cookie was empty, it would not have been possible to check the user's
        // settings with a simple !status['cookie-euccx'].
        var cookieData = 'cookies-enabled:';
        $('#cookie-tabs input').each(function () {
          if ($(this).prop('checked')) {
            plugin = $(this).attr('name').match(/-[^-]+$/)[0].substring(1);
            cookieData += plugin + '|';
          }
        });
        // ... set our cookie
        Drupal.euccx.setCookie(cookieData);
        // ... and reload the page so that the user gets some feedback of
        // something happening and if for example he enabled google analytics it
        // records the visit.
        location.reload();
      }
      catch (error) {
        // console.log(error)
      }
    });
  };

  // Check the cookies that are stored in the user's browser and a return an
  // object with all the relevant information.
  Drupal.euccx.getCookieStatus = function () {
    var status = {};
    var cookieName = (typeof eu_cookie_compliance_cookie_name === 'undefined' ||
      eu_cookie_compliance_cookie_name === '') ?
      'cookie-agreed' :
      eu_cookie_compliance_cookie_name;
    var cookieEUCC = Drupal.euccx.getCookie(cookieName);
    var cookieEUCCX = Drupal.euccx.getCookie('cookie-extras-agreed');

    status['cookie-eucc'] = cookieEUCC;
    status['cookie-euccx'] = cookieEUCCX;
    if (typeof cookieEUCCX === 'string') {
      var pluginNames = Drupal.euccx.getPluginNames();
      for (var i = 0; i < pluginNames.length; i++) {
        status[pluginNames[i]] = (cookieEUCCX.indexOf(pluginNames[i]) !== -1);
      }
    }
    return status;
  };

  // Unblocks all plugins that used yett
  Drupal.euccx.yettUnblockAll = function () {
    if (!window.yett) {
      return;
    }
    window.yett.unblock();
  };

  // Whitelist a specific set of scripts that were blocked via yett
  Drupal.euccx.yettUnblockPlugin = function (rules) {
    if (!window.yett) {
      return;
    }
    // This should work now after a modification was made to the yett library.
    // See: https://github.com/snipsco/yett/issues/11 for more info on what the
    // problem was and how it was solved in 0.1.8 version of yett.

    // Convert rules to RegExps
    var regexp_rules = rules.map(function(rule) {
      // Strip leading/trailing slash.
      rule = rule.replace(/^\/(.+)\/$/, "$1");
      return new RegExp(rule);
    });

    window.yett.unblock.apply(this, regexp_rules);
  };

  // Unblock all plugins
  Drupal.euccx.unblockAll = function () {
    var plugins = Drupal.euccx.getPluginNames();
    for (var i = 0; i < plugins.length; i++) {
      Drupal.euccx.unblockPlugin(plugins[i]);
    }
  };

  /**
   * Returns a unique list of all accepted categories plugin names.
   * If categories are not used as opt-in method, returns all plugins.
   *
   * @return []
   */
  Drupal.euccx.getAcceptedCategoriesPluginNames = function () {
    var plugins = [];
    var categoryPlugins;
    if (Drupal.euccx.usesCategories()) {
      // Opt-in by categories is used, we may only unblock all from allowed
      // categories:
      if(!Drupal.eu_cookie_compliance.hasAgreed()){
        // We can return early here, nothing accepted at all!
        return [];
      }
      var acceptedCategories = Drupal.eu_cookie_compliance.getAcceptedCategories();
      for (var a = 0; a < acceptedCategories.length; a++) {
        categoryPlugins = Drupal.euccx.getPluginNamesByOptInCategory(acceptedCategories[a]);
        if(categoryPlugins && categoryPlugins.length > 0){
          plugins = plugins.concat(categoryPlugins.filter(function(el) {
            return plugins.indexOf(el) === -1;
          }));
        }
      }
    } else {
      // Not using categories, all defined plugins are accepted:
      plugins = Drupal.euccx.getPluginNames();
    }
    return plugins;
  };

  /**
   * Return an array of the plugin names that are in a given category.
   *
   * @param optInCategory The machine name of the eucc opt-in category
   * @return []
   */
  Drupal.euccx.getPluginNamesByOptInCategory = function (optInCategory) {
    if (Drupal.settings.euccx && Drupal.settings.euccx.plugins) {
      var pluginNames = [];
      for (var key in Drupal.settings.euccx.plugins) {
        if ({}.hasOwnProperty.call(Drupal.settings.euccx.plugins, key)) {
           if (Drupal.settings.euccx.plugins[key].opt_in_category == optInCategory) {
            pluginNames.push(key);
          }
        }
      }
      return pluginNames;
    }
    return false;
  };

  // Given a plugin name execute the function that will unblock its code
  Drupal.euccx.unblockPlugin = function (pluginName) {
    var functionName = 'euccx' + pluginName + 'Load';
    try {
      window[functionName]();
    }
    catch (e) {
      // console.log(e);
    }
  };

  // Return an array of the plugin names that are defined
  Drupal.euccx.getPluginNames = function () {
    if (Drupal.settings.euccx && Drupal.settings.euccx.plugins) {
      var pluginNames = [];
      for (var key in Drupal.settings.euccx.plugins) {
        if ({}.hasOwnProperty.call(Drupal.settings.euccx.plugins, key)) {
          pluginNames.push(key);
        }
      }
      return pluginNames;
    }
    return false;
  };

  // Return false when the cookie is not properly set or the cookie value when
  // it is (properly set).
  Drupal.euccx.getCookie = function (cookieName) {
    var cookieValue = $.cookie(cookieName);
    if (typeof (cookieValue) === 'undefined' || (cookieValue === null)) {
      return false;
    }
    return cookieValue;
  };

  // This is using the exact same code with EU Cookie since our cookie is also
  // respecting most of the settings that the user supplied in EU Cookie
  // Compliance module.
  Drupal.euccx.setCookie = function (content) {
    try {
      var date = new Date();
      var domain = Drupal.settings.eu_cookie_compliance.domain ?
        Drupal.settings.eu_cookie_compliance.domain :
        '';
      var path = Drupal.settings.basePath;
      var cookieName = 'cookie-extras-agreed';
      if (path.length > 1) {
        var pathEnd = path.length - 1;
        if (path.lastIndexOf('/') === pathEnd) {
          path = path.substring(0, pathEnd);
        }
      }
      var cookieSession = parseInt(
        Drupal.settings.eu_cookie_compliance.cookie_session
      );

      if (cookieSession) {
        $.cookie(cookieName, content, {path: path, domain: domain});
      }
      else {
        var lifetime = parseInt(
          Drupal.settings.eu_cookie_compliance.cookie_lifetime
        );
        date.setDate(date.getDate() + lifetime);
        $.cookie(cookieName, content, {
          expires: date,
          path: path,
          domain: domain
        });
      }

      // Store consent if applicable.
      if (Drupal.settings.eu_cookie_compliance.store_consent) {
        var url = Drupal.settings.basePath;
        url += Drupal.settings.pathPrefix;
        url += 'eu-cookie-compliance/store_consent/banner';

        $.post(url, {}, function (data) { });
      }
    }
    catch (e) {
      // console.log(e);
    }
  };

  // The switches in the block are always set to off. It's up to us to turn them
  // on, if the cookie stored tells us so.
  Drupal.euccx.handleSwitches = function () {
    // If the cookie is not set, just turn them on by default
    var cookieValue = Drupal.euccx.getCookie('cookie-extras-agreed');
    // If the cookie has never been set, we need to see if the user selected to
    // have the switches turned on or off by default
    // see: https://www.drupal.org/project/euccx/issues/3092522
    // If the user has already accepted the cookies in general (aka from the EU
    // Cookie Compliance button) then there is no reason to respect the
    // unticked option. For more information on the reasoning behind that
    // see: https://www.drupal.org/project/euccx/issues/3109741
    if (!cookieValue) {
      if (Drupal.settings.euccx.unticked && !Drupal.eu_cookie_compliance.hasAgreed()) {
        $('#cookie-tabs input').prop('checked', false);
      }
      else {
        $('#cookie-tabs input').prop('checked', true);
      }
    }
    else {
      var plugins = Drupal.settings.euccx.plugins;
      for (var key in plugins) {
        if ({}.hasOwnProperty.call(plugins, key)) {
          var input = $("input[name*='" + key + "']");
          if (input.length && (cookieValue.indexOf(key) !== -1)) {
            input.prop('checked', true);
          }
        }
      }
    }
  };

  /**
   * Handles the switches if eucc opt-in with categories is used.
   * In this case the switches are simple indicatory for the eucc category
   * status.
   */
  Drupal.euccx.handleCategorySwitches = function () {
    // Open the cookie popup whenever a pseudo switch is clicked.
    $("#cookie-tabs .euccx-slider").click(Drupal.eu_cookie_compliance.toggleWithdrawBanner);

    // Mark plugins from allowed categories as active (passively):
    var plugins = Drupal.euccx.getAcceptedCategoriesPluginNames();
    for (var i = 0; i < plugins.length; i++) {
      var currentPlugin = plugins[i];
      var $aliasSlider = $("#cookie-tabs .euccx-slider.euccx-slider-" + currentPlugin);
      if ($aliasSlider.length > 0) {
        $aliasSlider.addClass('euccx-slider-checked');
      }
    }
  };

  // Check our settings for cookies that plugins declared as being offensive.
  // If the pluginName is empty, return offending cookies from all enabled
  // plugins.
  // Note: this function screens out non-enabled plugins.
  Drupal.euccx.getOffendingCookies = function (pluginName) {
    var offendingCookies = [];
    var settings = Drupal.settings.euccx;
    var pluginNames = Drupal.euccx.getPluginNames();
    for (var i = 0; i < pluginNames.length; i++) {
      var currentPlugin = pluginNames[i];
      var cookies_handled = false;
      if (settings['plugins'] && settings['plugins'][currentPlugin] &&
        settings['plugins'][currentPlugin]['cookies_handled']) {
        cookies_handled = settings['plugins'][currentPlugin]['cookies_handled'];
      }

      // The below check translates to: if the plugin uses the cookies_handled
      // key AND it's an array AND (either this function was called for all
      // plugins or the plugin that it was called for is the one we're
      // checking right now), add the offending cookies to our array.
      if (cookies_handled && cookies_handled.constructor === Array &&
        (pluginName === '' || pluginName === pluginNames[i])) {
        offendingCookies = offendingCookies.concat(cookies_handled);
      }
    }
    return offendingCookies;
  };

  // We remove the offending cookies by setting their expiration date to the
  // past and their content to "null".
  // This is very similar to the functionality provided by the EU Cookie
  // Compliance module. The core differences are:
  // 1) Since the new version of jquery.cookie that you get through
  // jquery_update does not support removeCookie but instead allows you to
  // remove cookies by setting their value to null, we are using this new method
  // as fallback and are also setting the expiration date to the past.
  // 2) Since some cookies may be registered in both the subdomain but also the
  // root domain, we are blanket-replacing the cookie names that were defined by
  // our plugins.
  Drupal.behaviors.euccxRemoveOffendingCookies = {
    attach: function (context, settings) {
      $(document).ready(function () {
        var status = Drupal.euccx.getCookieStatus();
        var offendingCookies = [];
        // Check if there is an EU Cookie Compliance cookie. If there is not,
        // all cookies defined in our plugins, should be considered offending
        // cookies because the user has not agreed to anything yet.
        if (!status['cookie-eucc']) {
          offendingCookies = Drupal.euccx.getOffendingCookies('');
        }
        // If there is no euccx cookies but there is an eucc cookie, it means
        // that the user agreed to everything. However, if there is an eucc and
        // an euccx cookie, we need to dig a little deeper.
        else if (status['cookie-euccx']) {
          var pluginNames = Drupal.euccx.getPluginNames();
          for (let i = 0; i < pluginNames.length; i++) {
            var currentPlugin = pluginNames[i];
            // We only push the offending cookies of the current plugin, if the
            // plugin name is absent from our cookie.
            if (status['cookie-euccx'].indexOf(currentPlugin) === -1) {
              offendingCookies =
                Drupal.euccx.getOffendingCookies(currentPlugin);
            }
          }
        }
        // Loop through any offending cookie names that we've found above
        for (let i = 0; i < offendingCookies.length; i++) {
          var hostname = window.location.hostname;
          var index = 0;
          var ctd = offendingCookies[i];
          while (hostname !== '') {
            // Check if jQuery.removeCookie is available:
            if (typeof $.removeCookie !== typeof undefined) {
              // jQuery.removeCookie exists!
              // Remove the cookie entirely:
              $.removeCookie(ctd, {
                path: '/',
                domain: '.' + hostname
              });
              $.removeCookie(ctd, {
                path: '/',
                domain: hostname
              });
            } else {
              // Fallback if jQuery.removeCookie is not available.
              // Set the cookies content to null.
              // Only set the value if the cookie exists, otherwise we'd
              // create a new empty cookie here:
              if ($.cookie(ctd) !== typeof undefined) {
                // For subdomain:
                $.cookie(ctd, null, {
                  expires: -5,
                  domain: '.' + hostname,
                  path: '/'}
                );
                // For domain:
                $.cookie(ctd, null, {
                  expires: -5,
                  domain: hostname,
                  path: '/'}
                );
              }
            }
            index = hostname.indexOf('.');
            hostname = (index === -1) ? '' : hostname.substring(index + 1);
          }
        }
      });
    }
  };
})(jQuery);
