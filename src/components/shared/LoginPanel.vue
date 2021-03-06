<template>
  <div class="loginContainer">
    <h1>Log in to {{ site }}</h1>
    <b-button
      class="logButton"
      variant="success"
      v-on:click="
        site === 'Twitter' ? handleTwitterLogin() : handleRedditLogin()
      "
      v-if="!loggedIn"
    >
      Click to login
    </b-button>
    <div id="login-panel-logout-button">
      <b-button
        class="logButton"
        variant="success"
        v-on:click="
          site === 'Twitter' ? handleTwitterLogout() : handleRedditLogout()
        "
        v-if="loggedIn"
      >
        Click to logout
      </b-button>
    </div>
    <b-tooltip
      target="login-panel-logout-button"
      triggers="hover"
      placement="bottom"
    >
      This will clear your saved settings!
    </b-tooltip>
    <span class="loginMessage" v-bind:class="{ loginError, loggedIn }">
      {{ loginMessage }}
    </span>
  </div>
</template>

<script>
/* eslint-disable @typescript-eslint/camelcase */
import { Component, Vue } from "vue-property-decorator";
import Twitter from "twitter-lite";
import axios from "axios";
import electron from "electron";
import store from "@/store/index";
import constants from "@/store/constants";
import twitterApi from "@/twitterSecrets";
import redditAPI from "@/redditSecrets";
import helpers from "@/util/helpers";

const LoginPanelProps = Vue.extend({
  props: {
    site: String
  }
});

@Component
export default class LoginPanel extends LoginPanelProps {
  loginError = false;

  loginMessage = "Not logged in!";

  get loggedIn() {
    if (this.site === "Twitter") {
      if (store.state.twitter[constants.TWITTER_LOGGED_IN]) {
        this.loginMessage = `Logged in to Twitter as @${
          store.state.twitter[constants.TWITTER_SCREEN_NAME]
        }`;
      }

      return store.state.twitter[constants.TWITTER_LOGGED_IN];
    }
    if (this.site === "Reddit") {
      if (store.state.reddit[constants.REDDIT_LOGGED_IN]) {
        this.loginMessage = `Logged in to Reddit as '${
          store.state.reddit[constants.REDDIT_USER_NAME]
        }'`;
      }

      return store.state.reddit[constants.REDDIT_LOGGED_IN];
    }

    return null;
  }

  handleTwitterLogin() {
    this.loginError = false;

    const client = new Twitter({
      consumer_key: twitterApi.consumer_key,
      consumer_secret: twitterApi.consumer_secret
    });
    let oauth_verifier;
    const { BrowserWindow } = electron.remote;
    const mainWindow = electron.remote.getCurrentWindow();
    const twitterApiWindow = new BrowserWindow({
      parent: mainWindow
    });

    const makeFollowUpRequest = initialResponse => {
      client
        .getAccessToken({
          key: initialResponse.oauth_token,
          secret: initialResponse.oauth_token_secret,
          verifier: oauth_verifier
        })
        .then(verificationResponse => {
          if (
            !verificationResponse.oauth_token ||
            !verificationResponse.oauth_token_secret
          ) {
            // login has failed, abort
            this.loginMessage = "Failed to login to Twitter!";
            this.loginError = true;
            throw Error(verificationResponse);
          }

          store.dispatch(constants.UPDATE_TWITTER_USER_KEYS, {
            consumer_key: twitterApi.consumer_key,
            consumer_secret: twitterApi.consumer_secret,
            access_token_key: verificationResponse.oauth_token,
            access_token_secret: verificationResponse.oauth_token_secret
          });
          store.dispatch(
            constants.UPDATE_TWITTER_USER_CLIENT,
            new Twitter(store.state.twitter[constants.TWITTER_USER_KEYS])
          );

          store.dispatch(
            constants.UPDATE_TWITTER_SCREEN_NAME,
            verificationResponse.screen_name
          );
          store.dispatch(
            constants.UPDATE_TWITTER_USER_ID,
            verificationResponse.user_id
          );

          helpers.twitterGatherAndSetItems({
            apiRoute: "statuses/user_timeline",
            itemArray: []
          });
          helpers.twitterGatherAndSetItems({
            apiRoute: "favorites/list",
            itemArray: []
          });

          store.dispatch(constants.LOGIN_TO_TWITTER);
          this.loginMessage = `Logged in to twitter as @${
            store.state.twitter[constants.TWITTER_SCREEN_NAME]
          }`;
          twitterApiWindow.close();
        })
        .catch(error => {
          // eslint-disable-next-line no-console
          console.error("Failed to login to twitter with error:", error);
          twitterApiWindow.close();
        });
    };

    client
      .getRequestToken("https://google.com")
      .then(initialResponse => {
        twitterApiWindow.loadURL(
          `https://api.twitter.com/oauth/authenticate?oauth_token=${initialResponse.oauth_token}`
        );
        twitterApiWindow.webContents.on("did-navigate", (event, url) => {
          if (url.indexOf("google") >= 0) {
            const searchParams = new URLSearchParams(url.slice(23));
            oauth_verifier = searchParams.get("oauth_verifier");

            makeFollowUpRequest(initialResponse);
          }
        });
      })
      .catch(error => {
        // eslint-disable-next-line no-console
        console.error(
          `Failed to load twitter api window with error: ${JSON.stringify(
            error
          )}s`
        );
      });
  }

  handleRedditLogin() {
    this.loginError = false;

    const { BrowserWindow } = electron.remote;
    const mainWindow = electron.remote.getCurrentWindow();
    const redditAPIWindow = new BrowserWindow({
      parent: mainWindow
    });

    // mysteriously, using just reddit.com gets some bizarre bug where all passwords are treated incorectly
    //   using the old.reddit.com works fine though
    redditAPIWindow.loadURL(
      `https://old.reddit.com/api/v1/authorize?client_id=${redditAPI.clientId}&response_type=code&state=randomString&redirect_uri=https://google.com&duration=permanent&scope=identity,history,read,edit`
    );

    redditAPIWindow.webContents.on("did-navigate", (event, url) => {
      if (url.indexOf("google") >= 0 && url.indexOf("reddit") === -1) {
        const searchParams = new URLSearchParams(url.slice(23));
        const oauthCode = searchParams.get("code");
        redditAPIWindow.close();

        axios
          .post(
            "https://www.reddit.com/api/v1/access_token",
            {},
            {
              params: {
                grant_type: "authorization_code",
                code: oauthCode,
                redirect_uri: "https://google.com"
              },
              auth: {
                username: redditAPI.clientId,
                password: ""
              }
            }
          )
          .then(response => {
            if (response?.data?.error === "invalid_request") {
              this.loginMessage = "Failed to login to Reddit!";
              this.loginError = true;
              throw Error("invalid request");
            }

            store.dispatch(constants.LOGIN_TO_REDDIT);
            store.dispatch(
              constants.UPDATE_REDDIT_ACCESS_TOKEN,
              response.data.access_token
            );
            store.dispatch(
              constants.UPDATE_REDDIT_REFRESH_TOKEN,
              response.data.refresh_token
            );

            helpers.stopRedditAccessTokenRefresh();
            helpers.refreshRedditAccessToken();

            helpers
              .makeRedditGetRequest("https://oauth.reddit.com/api/v1/me")
              .then(meData => {
                store.dispatch(constants.UPDATE_REDDIT_USER_NAME, meData.name);

                helpers.redditGatherAndSetItems();
              });
          })
          .catch(error => {
            console.error("Failed to login to reddit with error:", error);
            redditAPIWindow.close();
          });
      }
    });
  }

  handleTwitterLogout() {
    this.loginError = false;

    const { BrowserWindow } = electron.remote;
    const mainWindow = electron.remote.getCurrentWindow();
    // this removes the cookie that will auto login to twitter
    mainWindow.webContents.session.cookies.remove(
      "https://twitter.com",
      "auth_token"
    );

    this.loginMessage = "Not Logged In!";
    store.dispatch(constants.LOGOUT_OF_TWITTER);
    helpers.stopTwitterContentRefresh();
  }

  handleRedditLogout() {
    this.loginError = false;

    // TODO(NG): remove cookies that auto login to reddit
    const { BrowserWindow } = electron.remote;
    const mainWindow = electron.remote.getCurrentWindow();
    // this removes the cookie that will auto login to twitter
    mainWindow.webContents.session.cookies.remove(
      "https://reddit.com",
      "reddit_session"
    );

    this.loginMessage = "Not Logged In!";
    store.dispatch(constants.LOGOUT_OF_REDDIT);
    helpers.stopRedditAccessTokenRefresh();
  }
}
</script>

<style lang="scss">
.loginContainer {
  font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
    Ubuntu, "Helvetica Neue", sans-serif;
  display: flex;
  justify-content: space-around;
  flex-direction: column;
  align-items: center;

  border: 4mm ridge #218838;
  padding: 20px;
  margin-top: 10px;
}

.logButton {
  width: 150px;
}

.loginMessage {
  padding-top: 10px;
}

.loginError {
  color: #dc3545;
}

.loggedIn {
  color: #218838;
}
</style>
