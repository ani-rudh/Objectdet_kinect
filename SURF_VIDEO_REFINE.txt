#include "stdafx.h"
#include <Windows.h>
#include <Kinect.h>
#include <Kinect.Face.h>
#include <opencv2/core.hpp>
#include <opencv2/opencv.hpp>
#include <opencv2\highgui.hpp>
#include <opencv2\imgproc.hpp>
#include<opencv2\xfeatures2d.hpp>
#include<opencv2\xfeatures2d\nonfree.hpp>

using namespace cv;
using namespace std;
using namespace cv::xfeatures2d;


int main()
{
	VideoCapture cap(1);
	if (!cap.isOpened())
	{
		cout << "Error opening the camera" << endl;
		return -1;
	}

	// matrix to hold the sample and loaded as a grayscale image//
	Mat sample = imread("samplepicedited.jpg", CV_LOAD_IMAGE_GRAYSCALE);
	if (!sample.data)
	{
		cout << "Error opening the sample picture" << endl;
		return -1;
	}

	//SURF detector variables
	int minHess = 1500;
	vector<KeyPoint> kp_sample, kp_target;
	Mat desc_sample, desc_target;

	//instantiate the class SURF//
	Ptr<SURF> detector = SURF::create(minHess);
	detector->detect(sample, kp_sample);

	Ptr<SURF> extractor = SURF::create();
	extractor->compute(sample, kp_sample, desc_sample);

	//matcher to match the keypoints - flann//
	FlannBasedMatcher matcher;

	char key = 'a';
	Mat cam, cvtfeed;

	while (key != 'q' || key != 'Q')
	{
		// create variables for the camera feed and convert it to grayscale

		cap >> cam;
		cvtColor(cam, cvtfeed, CV_RGB2GRAY);

		detector->detect(cvtfeed, kp_target);
		extractor->compute(cvtfeed, kp_target, desc_target);

		vector<DMatch> matches; // vector of keypoint descriptors
		matcher.match(desc_sample, desc_target, matches);// match the key point decriptors and store the pair of matching points as a vector "matches"

		double max_dist = 0; double min_dist = 100;

		vector< DMatch > goodmatches;// vector to store the best matched descriptors

		for (int i = 0; i < desc_sample.rows; i++) // for the number of sample keypoints found
		{
			if (matches[i].distance < 3 * min_dist) // filter the points if it is more than 300 pixels away
			{
				goodmatches.push_back(matches[i]); // add the "good" matches to the vector (Dmatch)
			}
		}

		//draw the matches
		Mat img_matches;
		vector<DMatch> dummy;
		/*vector<DMatch> method;
		char mode = 'd';
		if (mode = 'n'){ method = dummy; }
		else method = goodmatches;*/

		drawMatches(sample, kp_sample, //sample image and its key pointa
			cvtfeed, kp_target,			//target and its key points
			/*goodmatchesdummy*/dummy, img_matches,	//vector of the matches and the target image 
			Scalar::all(-1),
			DrawMatchesFlags::NOT_DRAW_SINGLE_POINTS);

		//to mark the object detected
		vector<Point2f> smpl;
		vector<Point2f> trgt;

		for (int j = 0; j < goodmatches.size(); j++)
		{
			smpl.push_back(kp_sample[goodmatches[j].queryIdx].pt);
			trgt.push_back(kp_target[goodmatches[j].trainIdx].pt);
		}

		Mat H = findHomography(smpl, trgt, CV_RANSAC);

		vector<Point2f> sampl_corner(4);
		vector<Point2f> trgt_corner(4);
		sampl_corner[0] = Point(0, 0);
		sampl_corner[1] = Point(sample.cols, 0);
		sampl_corner[2] = Point(sample.cols, sample.rows);
		sampl_corner[3] = Point(0, sample.rows);

		if (!H.empty())
		{
			perspectiveTransform(sampl_corner, trgt_corner, H);
		}

		line(img_matches, trgt_corner[0] + Point2f(sample.cols, 0), trgt_corner[1] + Point2f(sample.cols, 0), Scalar(255, 200, 150), 4);
		line(img_matches, trgt_corner[1] + Point2f(sample.cols, 0), trgt_corner[2] + Point2f(sample.cols, 0), Scalar(255, 200, 150), 4);
		line(img_matches, trgt_corner[2] + Point2f(sample.cols, 0), trgt_corner[3] + Point2f(sample.cols, 0), Scalar(255, 200, 150), 4);
		line(img_matches, trgt_corner[3] + Point2f(sample.cols, 0), trgt_corner[0] + Point2f(sample.cols, 0), Scalar(255, 200, 150), 4);

		//show the matched objects//
		namedWindow("converted feed", 1);
		imshow("converted feed", cvtfeed);

		namedWindow("Matches", CV_WINDOW_NORMAL);
		imshow("Matches", img_matches);



		if (cvWaitKey(30) == VK_ESCAPE)
		{
			cout << "exiting......." << endl;
			break;
		}

		key = 'a';

	}

	cap.release();


}
