# Handshake
auto snapchat streak
package com.packageName.snapchatuiautomat;

import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;

import android.content.Context;
import android.content.Intent;
import android.os.RemoteException;
import android.support.test.InstrumentationRegistry;
import android.support.test.filters.SdkSuppress;
import android.support.test.runner.AndroidJUnit4;
import android.support.test.uiautomator.UiDevice;
import android.support.test.uiautomator.By;
import android.support.test.uiautomator.UiObject;
import android.support.test.uiautomator.UiObject2;
import android.support.test.uiautomator.UiObjectNotFoundException;
import android.support.test.uiautomator.UiScrollable;
import android.support.test.uiautomator.UiSelector;
import android.support.test.uiautomator.Until;
import android.util.Log;

import junit.framework.Assert;

import java.io.File;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

import static org.hamcrest.core.IsNull.notNullValue;
import static org.junit.Assert.assertThat;

@RunWith(AndroidJUnit4.class)
@SdkSuppress(minSdkVersion = 18)
public class SnapShatter {

    //The EXACT names of the people you are sending to, INCLUDING EMOJI
    private static final String[] sendToUsers = {"<NAME OF THE PERSON TO SEND TO","SECOND NAME,,,"};

    private static final String SNAPCHAT_PACKAGE = "com.snapchat.android";
    private static final String SNAPCHAT_PASSWORD = "<THE PASSWORD FOR AUTO LOGIN>";
    private static final int LAUNCH_TIMEOUT = 5000;
    private UiDevice mDevice;
    @Test
    public void startMainActivityFromHomeScreen() {
        // Initialize UiDevice instance
        mDevice = UiDevice.getInstance(InstrumentationRegistry.getInstrumentation());
        try {
            mDevice.wakeUp();
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        //Prepare our screenshots folder
        try {
            Runtime.getRuntime().exec("mkdir /sdcard/SnapShatterScreenshots");
        } catch (IOException e) {
            e.printStackTrace();
        }
        // Start from the home screen
        mDevice.pressHome();

        // Wait for launcher
        final String launcherPackage = mDevice.getLauncherPackageName();
        assertThat(launcherPackage, notNullValue());
        mDevice.wait(Until.hasObject(By.pkg(launcherPackage).depth(0)),LAUNCH_TIMEOUT);

        // Launch the app
        Context context = InstrumentationRegistry.getContext();
        final Intent intent = context.getPackageManager().getLaunchIntentForPackage(SNAPCHAT_PACKAGE);
        
        // Clear out any previous instances
        intent.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TASK);
        context.startActivity(intent);

        // Wait for the app to appear
        mDevice.wait(Until.hasObject(By.pkg(SNAPCHAT_PACKAGE).depth(0)),LAUNCH_TIMEOUT);

        //Look for the sign in button to see if we're login or not
        UiObject loginButton = mDevice.findObject(new UiSelector().resourceId("com.snapchat.android:id/login_and_signup_page_fragment_login_button"));
        if (loginButton.exists()) {
            Log.d("UITEST","At login!");
            doLogin();
        }else {
            Log.d("UITEST","Already signed in?");
        }
        doSendSnap();
        try {
            //Wait a while for all the snaps to send
            Thread.sleep(10*1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        readAllSnaps();
        try {
            mDevice.sleep();
        } catch (RemoteException e) {
            e.printStackTrace();
        }

    }

    @Test
    public void doLogin() {
        mDevice = UiDevice.getInstance(InstrumentationRegistry.getInstrumentation());
        UiObject loginButton = mDevice.findObject(new UiSelector().resourceId("com.snapchat.android:id/login_and_signup_page_fragment_login_button"));
        try {
            loginButton.click();
            //Type our password
            mDevice.findObject(new UiSelector().resourceId("com.snapchat.android:id/password_field")).setText(SNAPCHAT_PASSWORD);
            //smash login
            mDevice.findObject(new UiSelector().resourceId("com.snapchat.android:id/registration_nav_container")).click();

            //Wait for login
            if (mDevice.wait(Until.hasObject(By.text("Search")),60*1000)) {
                Log.d("UITEST","Logged in");
            }else {
                Assert.fail("Login failed");
            }
        } catch (UiObjectNotFoundException e) {
            Log.d("UITEST","Couldn't find button?!");
            e.printStackTrace();
            Assert.fail("No login button?");
        }
    }

    @Test
    public void doSendSnap() {
        mDevice = UiDevice.getInstance(InstrumentationRegistry.getInstrumentation());
        UiObject takePhotoButton = mDevice.findObject(new UiSelector().resourceId("com.snapchat.android:id/nueva_nav_default_camera_button"));
        try {
            takePhotoButton.click();
            Thread.sleep(3000);
            //Click the add text button
            mDevice.findObject(new UiSelector().resourceId("com.snapchat.android:id/caption_btn_primary_view")).click();
            //Set the caption text
            mDevice.findObject(new UiSelector().resourceId("com.snapchat.android:id/snap_caption_view")).setText("It's me, streakbot, and I'm here to keep a streak!");
            Thread.sleep(2000);
            //Dismiss keyboard
            mDevice.click(0,0);
            Thread.sleep(2000);
            //Click the send to
            mDevice.findObject(new UiSelector().resourceId("com.snapchat.android:id/send_to_button_hint_container")).click();
            //Wait till we make it to the send selector view
            if (mDevice.wait(Until.hasObject(By.text("My Story")),10*1000)) {
                Assert.assertTrue("Prepared snap",true);
                selectSnapSentTo();
            }else {
                Assert.fail("Couldn't make it to the send to screen");
            }
        } catch (UiObjectNotFoundException e) {
            Log.d("UITEST","Couldn't find button?!");
            e.printStackTrace();
            Assert.fail("Snap send error, not found");
        }catch (InterruptedException e){}

    }

    @Test
    public void selectSnapSentTo() {
        mDevice = UiDevice.getInstance(InstrumentationRegistry.getInstrumentation());
        try {
            UiScrollable scrollable = new UiScrollable(new UiSelector().scrollable(true));
            //Set the dead zone because if you start at the top of the screen you are touching the menu bar, canceling the drag. Make this bigger if you can't scroll up
            scrollable.setSwipeDeadZonePercentage(0.25);
            //For each recipient, find them in the list and then click their name
            for (String sendTo : sendToUsers) {
                scrollable.scrollTextIntoView(sendTo);
                mDevice.findObject(new UiSelector().textContains(sendTo)).click();
                scrollable.scrollToBeginning(100);
                Log.d("UITEST","Selected recipient: " + sendTo);
            }
            //Now send it off. This id was a pain in the ass to find since XML is broke in this view
            //had to set a debugger and inspect List<UiObject2> a = mDevice.findObjects(By.clickable(true));
            //A disgustingly complicated way to match both send buttons (labeled and unlabeled) which exsists for some reason
            //com.snapchat.android:id/send_to_bottom_panel_send_button
            //com.snapchat.android:id/sent_to_button_label_mode_view
            UiObject a = mDevice.findObject(new UiSelector().clickable(true).scrollable(false).resourceIdMatches("com.snapchat.android:id\\/sen[dt]_to_b[uo]tto[mn]_.*"));
            a.click();

            //resourceId("com.snapchat.android:id/send_to_bottom_panel_send_button")
        }catch (UiObjectNotFoundException e) {
            Log.d("UITEST","Couldn't find user send button?!");
            e.printStackTrace();
            Assert.fail("Selector error");
        }
    }

    @Test
    /**
     * Go through and read all the snaps. You must already be at the snap list (which you'll end up here after sending a snap, so that works)
     */
    public void readAllSnaps() {
        mDevice = UiDevice.getInstance(InstrumentationRegistry.getInstrumentation());
        UiScrollable scrollable = new UiScrollable(new UiSelector().scrollable(true));
        //Set the dead zone because if you start at the top of the screen you are touching the menu bar, canceling the drag. Make this bigger if you can't scroll up
        scrollable.setSwipeDeadZonePercentage(0.25);

        try {
            //For each recipient, find them in the list and then click their name
            for (String sendTo : sendToUsers) {
                //For testing: the routine doesn't work sending to yourself because in the send list you have a (me) tag but in the view list you don't
                sendTo = sendTo.replace(" (me)","");
                //Init the list scroller
                scrollable.scrollTextIntoView(sendTo);
                mDevice.findObject(new UiSelector().textContains(sendTo)).click();
                Log.d("UITEST", "Checking snaps for: " + sendTo);

                //So now that we've clicked on one of our target users, we have one of two case: 1) we don't have snaps and are taken to chat or 2) we do and we have to watch them
                if (mDevice.wait(Until.hasObject(By.text("Send a chat")),2*1000)) {
                    Log.d("UITEST", "No snaps for " + sendTo);
                    //We're at chat, hit back
                    mDevice.findObject(new UiSelector().resourceId("com.snapchat.android:id/header_button")).click();
                    //Wait until it's gone
                    mDevice.wait(Until.gone(By.text("Send a chat")),2000);
                }else {
                    //We're on snaps. Now we just wait a very long time until the snap view disappears.
                    while (mDevice.hasObject(By.res(SNAPCHAT_PACKAGE,"video_frame")) || mDevice.hasObject(By.res(SNAPCHAT_PACKAGE,"snap_image_view"))) {
                        Log.d("UITEST", "Found snap for: " + sendTo);
                        String cleanName = sendTo.replace(" ","_").replace("/","").replace("\"","");
                        mDevice.takeScreenshot(new File("/sdcard/SnapShatterScreenshots/Snap_" + cleanName + "_" + System.currentTimeMillis() / 1000L + ".png"));
                        mDevice.click(0,0);
                        try {
                            Thread.sleep(500);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }
                scrollable.scrollToBeginning(100);

            }
        }catch (UiObjectNotFoundException e) {
            Log.d("UITEST","Couldn't find user to read snaps from");
            e.printStackTrace();
            Assert.fail("Selector error");
        }

        List<UiObject2> a = mDevice.findObjects(By.clickable(true));
        //Get the countdown timer, we want to wait for it to go away.
        //UiObject snapCountdownView = mDevice.findObject(new UiSelector().resourceId("com.snapchat.android:id/multi_level_snap_view"));
    }
}
