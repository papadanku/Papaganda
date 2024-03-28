
Lucas-Kanade Optical Flow in HLSL
=================================

An optical flow algorithm estimates the motion between frames. Optical flow is essential in object detection, object recognition, motion estimation, video compression, and video effects.

This post covers an HLSL implementation of Lucas-Kanade optical flow.

Algorithm
---------

The pyramid LK algorithm consists of the following steps.

#. Build the current frame’s mipmap pyramid

   Encode the image into chromaticity with ``GetSphericalRG()``

#. Filter the current frame with a Gaussian blur
#. Set the initial motion vector to ``<0.0, 0.0>``
#. Compute optical flow from the smallest to largest pyramid level

   Propagate the optical flow at each level

#. Filter the optical flow with a Gaussian blur
#. Store the current frame for use in the next frame

.. note::

   The code contains **generic** functions, so you may need to change some parts of the code so it is compatible with your setup.

Source Code
-----------

.. code-block:: c

    float GetHalfMax()
    {
        // Get the Half format distribution of bits
        // Sign Exponent Significand
        // 0    00000    000000000
        const int SignBit = 0;
        const int ExponentBits = 5;
        const int SignificandBits = 10;

        const int Bias = -15;
        const int Exponent = exp2(ExponentBits);
        const int Significand = exp2(SignificandBits);

        const float MaxExponent = (Exponent - exp2(1)) + Bias;
        const float MaxSignificand = 1.0 + (((float)Significand - 1.0) / (float)Significand);

        return pow(-1, SignBit) * exp2(MaxExponent) * MaxSignificand;
    }

    // [-Half, Half] -> [-1.0, 1.0]
    float2 UnpackMotionVectors(float2 Half2)
    {
        return clamp(Half2 / GetHalfMax(), -1.0, 1.0);
    }

    // [-1.0, 1.0] -> [-Half, Half]
    float2 PackMotionVectors(float2 Half2)
    {
        return Half2 * GetHalfMax();
    }

    // [-1.0, 1.0] -> [Width, Height]
    float2 UnnormalizeMotionVectors(float2 Vectors, float2 ImageSize)
    {
        return Vectors / abs(ImageSize);
    }

    // [Width, Height] -> [-1.0, 1.0]
    float2 NormalizeMotionVectors(float2 Vectors, float2 ImageSize)
    {
        return clamp(Vectors * abs(ImageSize), -1.0, 1.0);
    }

    /*
        Lucas-Kanade optical flow with bilinear fetches
        ---
        Calculate Lucas-Kanade optical flow by solving (A^-1 * B)
        [A11 A12]^-1 [-B1] -> [ A11/D -A12/D] [-B1]
        [A21 A22]^-1 [-B2] -> [-A21/D  A22/D] [-B2]
        ---
        [ Ix^2/D -IxIy/D] [-IxIt]
        [-IxIy/D  Iy^2/D] [-IyIt]
    */

    float2 GetPixelPyLK
    (
        float2 MainTex,
        float2 Vectors,
        sampler2D SampleI0,
        sampler2D SampleI1
    )
    {
        // Initialize variables
        float4 WarpTex;
        float IxIx = 0.0;
        float IyIy = 0.0;
        float IxIy = 0.0;
        float IxIt = 0.0;
        float IyIt = 0.0;

        // Get required data to calculate main texel data
        const float Pi2 = acos(-1.0) * 2.0;

        // Unpack motion vectors
        Vectors = UnpackMotionVectors(Vectors);

        // Calculate main texel data (TexelSize, TexelLOD)
        WarpTex = float4(MainTex, MainTex + Vectors);

        // Get gradient information
        float4 TexIx = ddx(WarpTex);
        float4 TexIy = ddy(WarpTex);
        float2 PixelSize = abs(TexIx.xy) + abs(TexIy.xy);

        [loop] for(int i = 1; i < 4; ++i)
        {
            [loop] for(int j = 0; j < 4 * i; ++j)
            {
                float Shift = (Pi2 / (4.0 * float(i))) * float(j);
                float2 AngleShift = 0.0;
                sincos(Shift, AngleShift.x, AngleShift.y);
                AngleShift *= float(i);

                // Get temporal gradient
                float4 TexIT = WarpTex.xyzw + (AngleShift.xyxy * PixelSize.xyxy);
                float2 I0 = tex2Dgrad(SampleI0, TexIT.xy, TexIx.xy, TexIy.xy).rg;
                float2 I1 = tex2Dgrad(SampleI1, TexIT.zw, TexIx.zw, TexIy.zw).rg;
                float2 IT = I0 - I1;

                // Get spatial gradient
                float4 OffsetNS = AngleShift.xyxy + float4(0.0, -1.0, 0.0, 1.0);
                float4 OffsetEW = AngleShift.xyxy + float4(-1.0, 0.0, 1.0, 0.0);
                float4 NS = WarpTex.xyxy + (OffsetNS * PixelSize.xyxy);
                float4 EW = WarpTex.xyxy + (OffsetEW * PixelSize.xyxy);
                float2 N = tex2Dgrad(SampleI0, NS.xy, TexIx.xy, TexIy.xy).rg;
                float2 S = tex2Dgrad(SampleI0, NS.zw, TexIx.xy, TexIy.xy).rg;
                float2 E = tex2Dgrad(SampleI0, EW.xy, TexIx.xy, TexIy.xy).rg;
                float2 W = tex2Dgrad(SampleI0, EW.zw, TexIx.xy, TexIy.xy).rg;
                float2 Ix = E - W;
                float2 Iy = N - S;

                // IxIx = A11; IyIy = A22; IxIy = A12/A22
                IxIx += dot(Ix, Ix);
                IyIy += dot(Iy, Iy);
                IxIy += dot(Ix, Iy);

                // IxIt = B1; IyIt = B2
                IxIt += dot(Ix, IT);
                IyIt += dot(Iy, IT);
            }
        }

        /*
            Calculate Lucas-Kanade matrix
            ---
            [ Ix^2/D -IxIy/D] [-IxIt]
            [-IxIy/D  Iy^2/D] [-IyIt]
        */

        // Calculate A^-1 and B
        float D = determinant(float2x2(IxIx, IxIy, IxIy, IyIy));
        float2x2 A = float2x2(IyIy, -IxIy, -IxIy, IxIx) / D;
        float2 B = float2(-IxIt, -IyIt);

        // Calculate A^T*B
        float2 Flow = (D == 0.0) ? 0.0 : mul(B, A);

        // Propagate motion vectors to Half format
        Vectors += NormalizeMotionVectors(Flow, PixelSize);

        // Pack motion vectors to Half format
        return PackMotionVectors(Vectors);
    }
